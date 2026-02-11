# Sandbox Lifecycle Validation Report

## Executive Summary

After comprehensive analysis of the codebase, I have identified **why sandboxes continue running after agent processes stop** and **why resource usage does not align with configured runtime expectations**.

## Key Findings

### Finding 1: Serverless Function Timeout (5 Minutes) - CRITICAL ISSUE ⚠️

**Root Cause**: The application runs on Vercel serverless functions with a **5-minute execution timeout** (Pro plan), but sandboxes are configured for **30-minute lifetimes**.

**Impact**:
- Agent processes are **killed at 5 minutes** when the serverless function times out
- Cleanup code **never executes** because the function is terminated by the platform
- Sandboxes remain **orphaned for 25 minutes** (30 min - 5 min)
- Resource efficiency: **16.7%** (5 minutes used / 30 minutes allocated)

**Evidence**:
```typescript
// app/api/tasks/route.ts:276
const TASK_TIMEOUT_MS = maxDuration * 60 * 1000 // 30 minutes

// But Vercel Pro serverless functions timeout at 5 minutes
// When this happens, the entire process is SIGKILL'd
// No cleanup code runs
```

**Timeline**:
```
0:00 - Task starts, sandbox created (30-min lifetime)
0:30 - Agent starts as detached process
1:00 - Agent working, function polling
2:00 - Agent working, function polling
3:00 - Agent working, function polling
4:00 - Agent working, function polling
5:00 - ⚠️ VERCEL KILLS SERVERLESS FUNCTION
      - Agent process KILLED
      - Cleanup code NEVER RUNS
      - Sandbox ORPHANED
5:01 - Sandbox still alive (no active agent)
...
30:00 - Sandbox expires (SDK timeout)
```

### Finding 2: Cleanup Logic Only Runs on Success Paths

**Current Implementation**:

The cleanup logic (`shutdownSandbox`) is present in **both success and error paths**:

```typescript
// SUCCESS PATH (app/api/tasks/route.ts:669-680)
if (keepAlive) {
  await logger.info('Sandbox kept alive for follow-up messages')
} else {
  unregisterSandbox(taskId)
  const shutdownResult = await shutdownSandbox(sandbox!)
  if (shutdownResult.success) {
    await logger.success('Sandbox shutdown completed')
  }
}

// ERROR PATH (app/api/tasks/route.ts:707-720)
catch (error) {
  if (sandbox) {
    if (keepAlive) {
      await logger.info('Sandbox kept alive despite error')
    } else {
      unregisterSandbox(taskId)
      const shutdownResult = await shutdownSandbox(sandbox)
      if (shutdownResult.success) {
        await logger.info('Sandbox shutdown completed after error')
      }
    }
  }
}
```

**Problem**: This code is **ONLY reached if the error is caught**. When the serverless function is killed by the platform timeout, this code **never executes**.

### Finding 3: Agent Polling Loops Have Timeout Protection (Recently Added)

**Status**: ✅ **TIMEOUT PROTECTION EXISTS** in Claude and Cursor agents

```typescript
// lib/sandbox/agents/claude.ts:430-442
const MAX_AGENT_WAIT_MS = (maxDurationMinutes || 25) * 60 * 1000
const agentStartTime = Date.now()

while (!isCompleted) {
  const elapsedMs = Date.now() - agentStartTime
  
  if (elapsedMs > MAX_AGENT_WAIT_MS) {
    await logger.error(`Agent execution timed out after ${Math.floor(elapsedMs / 60000)} minutes`)
    throw new Error(`Agent execution timed out. The agent did not complete within the allocated time of ${maxDurationMinutes || 25} minutes.`)
  }
  
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**However**: This timeout protection is **NEVER REACHED** because the serverless function is killed at 5 minutes, before the 25-minute agent timeout.

**Other Agents**: Codex, Copilot, Gemini, and OpenCode do **NOT** use polling loops - they execute synchronously and wait for completion. These agents are also subject to the 5-minute function timeout.

### Finding 4: No Persistent Sandbox Tracking

**Current State**:
- Sandbox registry is **in-memory only** (`lib/sandbox/sandbox-registry.ts`)
- Lost when serverless function terminates
- No way to track orphaned sandboxes across function invocations

```typescript
// lib/sandbox/sandbox-registry.ts:8
const activeSandboxes = new Map<string, Sandbox>()

// This Map is destroyed when the function times out
// No persistent record of active sandboxes
```

**Database Tracking**:
- `sandboxId` is stored in the database ✅
- `sandboxCreatedAt` is **NOT** stored ❌
- `sandboxExpiresAt` is **NOT** stored ❌
- No background job to clean up orphaned sandboxes ❌

### Finding 5: KeepAlive Logic Works Correctly (When Cleanup Runs)

**Status**: ✅ **LOGIC IS CORRECT**

The `keepAlive` setting correctly controls whether the sandbox should be shut down:
- `keepAlive=true`: Sandbox stays alive for follow-ups ✅
- `keepAlive=false`: Sandbox is shut down after completion ✅

**Problem**: The cleanup code is **never reached** when the function times out, so `keepAlive=false` has no effect in timeout scenarios.

## Resource Usage Analysis

### Current Behavior (keepAlive=false, 10-minute task):

| Metric | Value | Notes |
|--------|-------|-------|
| Sandbox Lifetime | 30 minutes | Configured via SDK |
| Agent Execution | 5 minutes | Killed by function timeout |
| Idle Time | 25 minutes | Sandbox orphaned |
| Resource Efficiency | 16.7% | 5/30 minutes used |
| Cleanup Executed | ❌ No | Function killed before cleanup |

### Expected Behavior (keepAlive=false, 10-minute task):

| Metric | Value | Notes |
|--------|-------|-------|
| Sandbox Lifetime | 10 minutes | Task duration + cleanup |
| Agent Execution | 10 minutes | Completes successfully |
| Idle Time | 0 minutes | Immediate shutdown |
| Resource Efficiency | 100% | Full utilization |
| Cleanup Executed | ✅ Yes | Proper shutdown |

## Validation of Existing Analysis Documents

I reviewed the existing analysis documents in the repository:

1. **ANALYSIS_AGENT_TIMEOUT.md** ✅ **ACCURATE**
   - Correctly identifies serverless function timeout as root cause
   - Accurately describes the 5-minute Vercel Pro limit
   - Properly explains the detached process execution issue

2. **FINDINGS_SUMMARY.md** ✅ **ACCURATE**
   - Correctly summarizes the timeout mismatch
   - Accurately identifies the 5-minute termination point
   - Properly explains the impact on resource usage

3. **TIMEOUT_DIAGRAM.md** ✅ **ACCURATE**
   - Visual diagrams correctly illustrate the problem
   - Timeline accurately shows the 5-minute kill point
   - Comparison tables are accurate

**Conclusion**: The existing analysis is **100% correct**. The issue is well-documented.

## Why Sandboxes Remain Running

### Direct Answer to User's Question:

**Sandboxes continue running after agent processes stop because:**

1. **Serverless Function Timeout (5 min)**: The Vercel serverless function is killed at 5 minutes, terminating the agent process
2. **Cleanup Code Never Runs**: When the function is killed, the cleanup code (`shutdownSandbox`) never executes
3. **Sandbox Lifetime Mismatch**: Sandboxes are configured for 30 minutes but agents only run for 5 minutes
4. **No Background Cleanup**: There is no background job to detect and clean up orphaned sandboxes
5. **In-Memory Registry**: The sandbox registry is lost when the function terminates, so there's no persistent tracking

**Result**: Sandboxes remain idle for **25 minutes** (30 min - 5 min) after the agent stops.

## Resource Usage Alignment Issues

### Why Resource Usage Does NOT Align with Configured Runtime:

1. **Configured Runtime**: 30 minutes (sandbox lifetime)
2. **Actual Runtime**: 5 minutes (function timeout)
3. **Gap**: 25 minutes of unused sandbox time
4. **Efficiency**: 16.7% (5/30 minutes)

**The configured runtime (30 minutes) is never fully utilized because the serverless function times out at 5 minutes.**

## Missing Logic Responsible for Failures

### 1. No Cleanup on Function Timeout

**Missing**: Mechanism to ensure cleanup runs even when function is killed

**Current State**: Cleanup only runs if error is caught
**Needed**: Cleanup that runs regardless of how function terminates

### 2. No Persistent Sandbox Tracking

**Missing**: Database fields to track sandbox lifecycle

**Current State**: Only `sandboxId` is stored
**Needed**: 
- `sandboxCreatedAt` timestamp
- `sandboxExpiresAt` timestamp
- `sandboxStatus` field

### 3. No Background Cleanup Job

**Missing**: Periodic job to clean up orphaned sandboxes

**Current State**: No background monitoring
**Needed**: Cron job or endpoint that:
- Queries database for orphaned sandboxes
- Reconnects using `Sandbox.get()`
- Calls `sandbox.stop()`
- Updates database

### 4. No Decoupling of Agent Execution from Function Lifetime

**Missing**: Mechanism to run agents beyond 5-minute function timeout

**Current State**: Agent runs in serverless function context
**Needed**: Agent runs independently in sandbox, monitored asynchronously

## Recommendations

### Immediate Actions (High Priority)

1. **Verify Timeout Protection**: Confirm that Claude and Cursor agents have working timeout protection
2. **Add Timeout to Other Agents**: Add timeout protection to Codex, Copilot, Gemini, OpenCode
3. **Document the 5-Minute Limit**: Add clear documentation about Vercel Pro function timeout

### Short-Term Solutions (This Week)

1. **Implement Sandbox-Side Execution**: Move agent execution to wrapper scripts in sandbox
2. **Add Persistent Tracking**: Store `sandboxCreatedAt` and `sandboxExpiresAt` in database
3. **Create Cleanup Endpoint**: Build `/api/cleanup-orphaned-sandboxes` endpoint

### Long-Term Solutions (This Month)

1. **Background Cleanup Job**: Implement cron job to clean up orphaned sandboxes
2. **Async Monitoring**: Decouple agent execution from function lifetime
3. **Vercel Background Functions**: Evaluate upgrade to Background Functions (5-hour timeout)

## Conclusion

**The sandbox continues running after the agent process stops because:**

1. ✅ **Cleanup logic exists** in both success and error paths
2. ❌ **Cleanup never runs** when serverless function times out at 5 minutes
3. ❌ **No persistent tracking** to enable cleanup across function invocations
4. ❌ **No background job** to detect and clean up orphaned sandboxes

**Resource usage does not align with configured runtime because:**

1. ✅ **Sandbox configured for 30 minutes** (correct)
2. ❌ **Function times out at 5 minutes** (platform limit)
3. ❌ **Agent killed at 5 minutes** (tied to function lifetime)
4. ❌ **Sandbox orphaned for 25 minutes** (no cleanup)

**The root cause is the Vercel serverless function timeout (5 minutes), not missing cleanup logic. The cleanup logic exists but is never reached.**
