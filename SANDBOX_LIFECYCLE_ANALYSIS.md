# Sandbox Lifecycle Analysis - Idle Sandbox Issue

## Problem Statement

Sandboxes continue running after the agent process stops, even when `keepAlive=false`. This results in:
- Idle sandboxes consuming resources without active agents
- Resource usage not aligned with configured runtime expectations
- Potential cost implications from unused sandbox time

## Root Cause Analysis

### 1. **Sandbox Lifecycle Mismatch**

The sandbox has TWO separate timeout mechanisms that are not properly coordinated:

#### A. Sandbox Runtime Timeout (Vercel SDK)
- **Location**: `lib/sandbox/creation.ts:77`
- **Configuration**: `timeout: timeoutMs` (e.g., 30 minutes)
- **Behavior**: Sandbox automatically expires after this duration
- **Status**: ✅ Working as designed

#### B. Agent Execution Timeout (Application Logic)
- **Location**: `lib/sandbox/agents/claude.ts:430`, `cursor.ts:481`
- **Configuration**: `MAX_AGENT_WAIT_MS` (25 minutes default)
- **Behavior**: Infinite polling loop waiting for agent completion
- **Status**: ⚠️ **NO TIMEOUT PROTECTION** (recently added but not enforced)

### 2. **KeepAlive Logic Issues**

The `keepAlive` setting controls whether the sandbox should be shut down after task completion:

#### Current Implementation:
```typescript
// app/api/tasks/route.ts:669-680
if (keepAlive) {
  await logger.info('Sandbox kept alive for follow-up messages')
} else {
  unregisterSandbox(taskId)
  const shutdownResult = await shutdownSandbox(sandbox!)
  if (shutdownResult.success) {
    await logger.success('Sandbox shutdown completed')
  }
}
```

#### Problem:
**This code is ONLY executed when the agent completes successfully.** If:
- Agent times out
- Agent crashes
- Serverless function is killed
- Error occurs during execution

The sandbox is **NEVER shut down** even when `keepAlive=false`.

### 3. **Serverless Function Timeout (5 Minutes)**

The most critical issue is the **Vercel serverless function timeout**:

#### Timeline:
```
0:00 - Task starts, sandbox created (30-min lifetime)
0:30 - Agent CLI starts as detached process
1:00 - Agent working, function polling
2:00 - Agent working, function polling
3:00 - Agent working, function polling
4:00 - Agent working, function polling
5:00 - ⚠️ SERVERLESS FUNCTION TIMEOUT (Vercel Pro: 5 min)
      - Function process KILLED
      - Agent process KILLED
      - Polling loop TERMINATED
      - Cleanup code NEVER RUNS
5:01 - Sandbox still alive (orphaned)
...
30:00 - Sandbox expires (SDK timeout)
```

#### Evidence:
- **Vercel Pro Plan**: 5-minute serverless function timeout
- **Function execution**: Uses `after()` but still subject to platform limits
- **Agent execution**: Runs as detached process, tied to function lifetime
- **Result**: Function killed at 5 minutes, sandbox orphaned for 25 minutes

### 4. **Missing Cleanup Paths**

The cleanup logic (`shutdownSandbox`) is only called in **success paths**:

#### Success Path (✅ Cleanup happens):
```
processTask() 
  → executeAgentInSandbox() succeeds
  → pushChangesToBranch()
  → if (!keepAlive) shutdownSandbox()
```

#### Error Paths (❌ Cleanup may not happen):
```
processTask()
  → executeAgentInSandbox() fails
  → catch block
  → if (!keepAlive) shutdownSandbox()  // ✅ Present
  
processTask()
  → Serverless function timeout (5 min)
  → Process KILLED
  → No cleanup code runs  // ❌ MISSING
```

## Identified Issues

### Issue 1: No Timeout Protection in Agent Polling Loops
**Files**: 
- `lib/sandbox/agents/claude.ts:430-450`
- `lib/sandbox/agents/cursor.ts:481-502`

**Current Code**:
```typescript
while (!isCompleted) {
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**Problem**: Infinite loop with no timeout enforcement

**Recent Addition** (not enforced):
```typescript
const MAX_AGENT_WAIT_MS = (maxDurationMinutes || 25) * 60 * 1000
const agentStartTime = Date.now()

while (!isCompleted) {
  const elapsedMs = Date.now() - agentStartTime
  if (elapsedMs > MAX_AGENT_WAIT_MS) {
    throw new Error(`Agent execution timed out...`)
  }
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**Status**: Code exists but may not be reached due to serverless timeout

### Issue 2: Serverless Function Timeout Kills Process Before Cleanup
**File**: `app/api/tasks/route.ts`

**Problem**: 
- Function has 5-minute timeout (Vercel Pro)
- Sandbox has 30-minute timeout
- Agent can run for 25+ minutes
- When function times out, cleanup never runs

**Impact**: Sandbox orphaned for remaining duration (up to 25 minutes)

### Issue 3: KeepAlive=false Not Enforced on Errors
**Files**: 
- `app/api/tasks/route.ts:710-720`
- `app/api/tasks/[taskId]/continue/route.ts:445-455`

**Current Error Handling**:
```typescript
catch (error) {
  if (sandbox) {
    if (keepAlive) {
      await logger.info('Sandbox kept alive despite error')
    } else {
      unregisterSandbox(taskId)
      const shutdownResult = await shutdownSandbox(sandbox)
    }
  }
}
```

**Problem**: This code is only reached if error is caught. If function is killed by platform timeout, this never runs.

### Issue 4: Sandbox Registry Only Tracks In-Memory State
**File**: `lib/sandbox/sandbox-registry.ts`

**Current Implementation**:
```typescript
const activeSandboxes = new Map<string, Sandbox>()

export function registerSandbox(taskId: string, sandbox: Sandbox, _keepAlive: boolean = false): void {
  activeSandboxes.set(taskId, sandbox)
}
```

**Problem**: 
- Registry is in-memory only
- Lost when serverless function terminates
- No persistent tracking of active sandboxes
- Cannot clean up orphaned sandboxes across function invocations

### Issue 5: No Sandbox Health Monitoring
**Missing**: Background process to monitor and clean up idle sandboxes

**Current State**:
- Sandboxes created with `sandboxId` stored in database
- No periodic check for idle sandboxes
- No automatic cleanup of orphaned sandboxes
- Relies entirely on SDK timeout (30 minutes)

## Resource Usage Analysis

### Current Behavior (keepAlive=false, 10-minute task):

```
Sandbox Lifetime: 30 minutes (configured)
Agent Execution: 5 minutes (killed by function timeout)
Idle Time: 25 minutes (sandbox orphaned)
Resource Efficiency: 16.7% (5/30 minutes used)
```

### Expected Behavior (keepAlive=false, 10-minute task):

```
Sandbox Lifetime: 10 minutes (task duration + cleanup)
Agent Execution: 10 minutes
Idle Time: 0 minutes
Resource Efficiency: 100%
```

### Current Behavior (keepAlive=true, 10-minute task):

```
Sandbox Lifetime: 30 minutes (configured)
Agent Execution: 5 minutes (killed by function timeout)
Idle Time: 25 minutes (waiting for follow-ups that never come)
Resource Efficiency: 16.7%
```

## Proposed Solutions

### Solution 1: Add Timeout Protection to Agent Loops (Immediate)
**Priority**: HIGH
**Effort**: LOW
**Impact**: Prevents infinite loops, provides clear errors

**Implementation**:
```typescript
// lib/sandbox/agents/claude.ts
const MAX_AGENT_WAIT_MS = (maxDurationMinutes || 25) * 60 * 1000
const agentStartTime = Date.now()
let lastLogTime = Date.now()

while (!isCompleted) {
  const elapsedMs = Date.now() - agentStartTime
  
  // Enforce timeout
  if (elapsedMs > MAX_AGENT_WAIT_MS) {
    await logger.error(`Agent execution timed out after ${Math.floor(elapsedMs / 60000)} minutes`)
    throw new Error(`Agent execution timed out. The agent did not complete within the allocated time of ${maxDurationMinutes || 25} minutes.`)
  }
  
  // Log progress every 30 seconds
  if (Date.now() - lastLogTime > 30000) {
    await logger.info(`Agent still running... (${Math.floor(elapsedMs / 60000)} minutes elapsed)`)
    lastLogTime = Date.now()
  }
  
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**Status**: ✅ Code already exists in claude.ts and cursor.ts, needs verification

### Solution 2: Decouple Agent Execution from Serverless Function (Critical)
**Priority**: CRITICAL
**Effort**: MEDIUM
**Impact**: Allows agents to run beyond 5-minute function timeout

**Implementation**:
```typescript
// Create wrapper script in sandbox
const wrapperScript = `#!/bin/bash
set -e
cd ${PROJECT_DIR}

# Run agent with timeout
timeout ${maxDuration}m ${agentCommand} > /tmp/agent-output.log 2>&1
EXIT_CODE=$?

# Save exit code
echo $EXIT_CODE > /tmp/agent-exit-code

# If successful, commit and push
if [ $EXIT_CODE -eq 0 ]; then
  git add .
  git commit -m "${commitMessage}" || true
  git push origin ${branchName}
fi

# Signal completion
echo "COMPLETED" > /tmp/agent-status
`

// Execute wrapper in sandbox
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', wrapperScript],
  detached: true,
  cwd: PROJECT_DIR,
})

// Exit function immediately
await logger.info('Agent started in sandbox, monitoring asynchronously')
return { success: true, message: 'Agent running in background' }
```

**Monitoring**:
```typescript
// Separate endpoint: GET /api/tasks/[taskId]/status
// Polls /tmp/agent-status every 10 seconds
// Updates task status in database
```

### Solution 3: Implement Sandbox Cleanup on Function Exit (Immediate)
**Priority**: HIGH
**Effort**: LOW
**Impact**: Ensures cleanup even when function times out

**Implementation**:
```typescript
// app/api/tasks/route.ts
after(async () => {
  try {
    await processTaskWithTimeout(...)
  } finally {
    // ALWAYS attempt cleanup
    if (sandbox && !keepAlive) {
      try {
        await shutdownSandbox(sandbox)
        await logger.info('Sandbox cleanup completed')
      } catch (error) {
        console.error('Sandbox cleanup failed:', error)
      }
    }
  }
})
```

**Problem**: This still won't work if function is killed by platform timeout

### Solution 4: Persistent Sandbox Tracking (Long-term)
**Priority**: MEDIUM
**Effort**: MEDIUM
**Impact**: Enables cleanup of orphaned sandboxes

**Implementation**:
```typescript
// Store sandbox state in database
await db.update(tasks).set({
  sandboxId: sandbox.sandboxId,
  sandboxCreatedAt: new Date(),
  sandboxExpiresAt: new Date(Date.now() + maxDuration * 60 * 1000),
  keepAlive: keepAlive,
}).where(eq(tasks.id, taskId))

// Background job (cron or separate endpoint)
// Runs every 5 minutes
async function cleanupOrphanedSandboxes() {
  const orphanedTasks = await db.select()
    .from(tasks)
    .where(and(
      isNotNull(tasks.sandboxId),
      eq(tasks.keepAlive, false),
      eq(tasks.status, 'completed'),
      // Sandbox created more than 10 minutes ago
      lt(tasks.sandboxCreatedAt, new Date(Date.now() - 10 * 60 * 1000))
    ))
  
  for (const task of orphanedTasks) {
    try {
      const sandbox = await Sandbox.get({
        sandboxId: task.sandboxId,
        teamId: process.env.SANDBOX_VERCEL_TEAM_ID!,
        projectId: process.env.SANDBOX_VERCEL_PROJECT_ID!,
        token: process.env.SANDBOX_VERCEL_TOKEN!,
      })
      
      if (sandbox) {
        await sandbox.stop()
        console.log(`Cleaned up orphaned sandbox: ${task.sandboxId}`)
      }
    } catch (error) {
      console.error(`Failed to cleanup sandbox ${task.sandboxId}:`, error)
    }
  }
}
```

### Solution 5: Reduce Sandbox Timeout to Match Agent Timeout (Quick Fix)
**Priority**: LOW
**Effort**: LOW
**Impact**: Reduces idle time but doesn't solve root cause

**Implementation**:
```typescript
// lib/sandbox/creation.ts
// Instead of fixed 30 minutes, use maxDuration + buffer
const timeoutMs = (config.timeout ? parseInt(config.timeout.replace(/\D/g, '')) + 5) * 60 * 1000

const sandboxConfig = {
  timeout: timeoutMs, // e.g., 30 minutes for 25-minute agent
  // ...
}
```

**Benefit**: Reduces wasted sandbox time from 25 minutes to 5 minutes
**Limitation**: Doesn't solve the serverless function timeout issue

## Recommended Implementation Plan

### Phase 1: Immediate Fixes (Today)
1. ✅ Verify timeout protection in agent polling loops (claude.ts, cursor.ts)
2. ✅ Add timeout protection to other agents (codex, copilot, gemini, opencode)
3. ✅ Add better error logging for timeout scenarios
4. ✅ Test with 6-minute task to confirm function timeout behavior

### Phase 2: Critical Fix (This Week)
1. ✅ Implement Solution 2: Decouple agent execution from serverless function
2. ✅ Create sandbox-side wrapper scripts for each agent
3. ✅ Implement async monitoring endpoint
4. ✅ Test with 10-minute and 30-minute tasks

### Phase 3: Long-term Improvements (This Month)
1. ✅ Implement Solution 4: Persistent sandbox tracking
2. ✅ Create background cleanup job
3. ✅ Add sandbox health monitoring dashboard
4. ✅ Optimize sandbox timeout based on actual usage

## Testing Plan

### Test Case 1: Short Task (< 5 minutes)
- **Setup**: keepAlive=false, 3-minute task
- **Expected**: Agent completes, sandbox shut down immediately
- **Verify**: No idle sandbox time

### Test Case 2: Medium Task (5-10 minutes)
- **Setup**: keepAlive=false, 7-minute task
- **Expected**: Agent completes beyond function timeout, sandbox shut down after completion
- **Verify**: Sandbox runs for ~7 minutes, not 30 minutes

### Test Case 3: Long Task (> 10 minutes)
- **Setup**: keepAlive=false, 15-minute task
- **Expected**: Agent completes, sandbox shut down after completion
- **Verify**: Sandbox runs for ~15 minutes, not 30 minutes

### Test Case 4: KeepAlive Enabled
- **Setup**: keepAlive=true, 5-minute task
- **Expected**: Agent completes, sandbox stays alive for follow-ups
- **Verify**: Sandbox runs for full 30 minutes or until manually stopped

### Test Case 5: Agent Timeout
- **Setup**: keepAlive=false, agent hangs indefinitely
- **Expected**: Agent times out at maxDuration, sandbox shut down
- **Verify**: Clear timeout error, sandbox cleaned up

### Test Case 6: Orphaned Sandbox Cleanup
- **Setup**: Create task, kill function manually, wait 10 minutes
- **Expected**: Background job detects orphaned sandbox and cleans it up
- **Verify**: Sandbox stopped within 10 minutes of orphaning

## Metrics to Monitor

1. **Sandbox Utilization Rate**: `(agent_execution_time / sandbox_lifetime) * 100`
   - Target: > 80% for keepAlive=false
   - Current: ~16.7%

2. **Idle Sandbox Time**: `sandbox_lifetime - agent_execution_time`
   - Target: < 5 minutes for keepAlive=false
   - Current: ~25 minutes

3. **Orphaned Sandboxes**: Count of sandboxes running without active agents
   - Target: 0
   - Current: Unknown (no monitoring)

4. **Cleanup Success Rate**: `(successful_cleanups / total_tasks) * 100`
   - Target: > 95%
   - Current: Unknown

## Conclusion

The root cause of idle sandboxes is a **combination of three issues**:

1. **Serverless function timeout (5 min)** kills the process before cleanup can run
2. **Agent execution tied to function lifetime** means agents can't run beyond 5 minutes
3. **No persistent sandbox tracking** means orphaned sandboxes can't be cleaned up

**Immediate action required**: Implement Solution 2 (decouple agent execution) to allow agents to run beyond 5 minutes and ensure proper cleanup.

**Long-term action required**: Implement Solution 4 (persistent tracking) to enable cleanup of orphaned sandboxes across function invocations.
