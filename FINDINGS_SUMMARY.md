# Agent Termination Analysis - Key Findings

## Problem Statement
Agent processes are being terminated around the 5-minute mark despite the sandbox runtime being configured for 30 minutes.

## Root Cause
**Vercel Serverless Function Timeout (5 minutes on Pro plan)**

The issue is NOT with the sandbox configuration, but with the execution environment:

### 1. Sandbox Configuration ✅ CORRECT
- **Location:** `lib/sandbox/creation.ts` line 77
- **Configuration:** `timeout: timeoutMs` (correctly set to 30 minutes)
- **Status:** Working as intended

### 2. Agent Execution ❌ PROBLEMATIC
- **Location:** `lib/sandbox/agents/claude.ts` lines 415-430
- **Issue:** Agent CLI runs as detached process with infinite polling loop
- **Code:**
  ```typescript
  await sandbox.runCommand({
    detached: true,  // Runs in background
    // ...
  })
  
  while (!isCompleted) {  // INFINITE LOOP - no timeout
    await new Promise((resolve) => setTimeout(resolve, 1000))
  }
  ```

### 3. Serverless Function Timeout ❌ LIMITING FACTOR
- **Location:** `app/api/tasks/route.ts` (Next.js API route)
- **Vercel Pro Plan Limit:** 5 minutes (300 seconds)
- **Impact:** Function is killed at 5 minutes, terminating all child processes

## Why 5 Minutes Specifically?

| Plan | Timeout | Your Case |
|------|---------|-----------|
| Hobby | 10s | ❌ |
| **Pro** | **5 min** | **✅ MATCHES** |
| Enterprise | 15 min | ❌ |

The 5-minute termination aligns perfectly with **Vercel Pro plan's serverless function timeout**.

## Technical Flow

```
1. User creates task → Next.js API route handler starts
2. API route creates sandbox (30-min lifetime) ✅
3. API route executes agent CLI as detached process
4. API route enters infinite polling loop waiting for completion
5. ⏰ 5 minutes pass → Vercel kills the serverless function
6. ❌ Agent process terminated (SIGKILL)
7. ❌ Polling loop terminated
8. ❌ Task marked as failed/incomplete
```

## Evidence

### 1. Detached Process Pattern
Found in multiple agent implementations:
- `lib/sandbox/agents/claude.ts:419` - `detached: true`
- `lib/sandbox/agents/cursor.ts:468` - `detached: true`

### 2. Infinite Polling Loops
Both Claude and Cursor agents use:
```typescript
while (!isCompleted) {
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```
**No timeout mechanism exists in these loops.**

### 3. Sandbox Timeout Mismatch
- Sandbox configured: 30 minutes
- Function timeout: 5 minutes
- **Gap:** 25 minutes of unused sandbox time

## Impact

### Current Behavior
- ✅ Tasks < 5 minutes: Complete successfully
- ❌ Tasks > 5 minutes: Terminated prematurely
- ❌ Long-running agents: Never complete
- ❌ Sandbox resources: Wasted (30-min allocation, 5-min usage)

### User Experience
- Tasks appear to hang around 5 minutes
- No clear error message (hard kill)
- Sandbox remains alive but agent is dead
- Confusion about timeout settings

## Quick Fixes (Immediate)

### Fix 1: Add Timeout to Polling Loop
**File:** `lib/sandbox/agents/claude.ts` (and similar for other agents)

```typescript
// BEFORE
while (!isCompleted) {
  await new Promise((resolve) => setTimeout(resolve, 1000))
}

// AFTER
const MAX_WAIT_MS = maxDuration * 60 * 1000
const startTime = Date.now()

while (!isCompleted) {
  if (Date.now() - startTime > MAX_WAIT_MS) {
    throw new Error(`Agent timed out after ${maxDuration} minutes`)
  }
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**Impact:** Prevents infinite loops, provides clear error messages

### Fix 2: Early Function Exit
**File:** `app/api/tasks/route.ts`

```typescript
// Start agent in sandbox
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', `nohup ${fullCommand} > /tmp/agent.log 2>&1 &`],
  detached: true,
})

// Exit function immediately, let sandbox continue
await logger.info('Agent started, running in background')
return { success: true }

// Monitor via separate polling endpoint
```

**Impact:** Function exits before 5-min timeout, agent continues in sandbox

## Long-Term Solutions

### Option 1: Sandbox-Side Execution
Move agent orchestration entirely into the sandbox:
- Create wrapper script in sandbox
- Use `timeout` command for hard limits
- Poll status via file checks (less frequent)
- Function exits early

### Option 2: Vercel Background Functions
Upgrade to Vercel Background Functions:
- Execution time: up to 5 hours
- Designed for long-running tasks
- Requires Enterprise plan or add-on

### Option 3: Heartbeat Monitoring
Implement asynchronous monitoring:
- Agent runs independently in sandbox
- Separate endpoint polls for status
- Database tracks progress
- No function timeout issues

## Recommended Action Plan

### Immediate (Today)
1. ✅ Add timeout to polling loops (Fix 1)
2. ✅ Add better error logging
3. ✅ Deploy and test with 10-minute task

### Short-Term (This Week)
1. Implement early function exit (Fix 2)
2. Create sandbox-side wrapper scripts
3. Test with 30-minute tasks

### Long-Term (This Month)
1. Evaluate Vercel Background Functions
2. Implement heartbeat monitoring system
3. Migrate to fully asynchronous architecture

## Testing Checklist

- [ ] Test task < 5 minutes (should work)
- [ ] Test task = 6 minutes (should fail currently)
- [ ] Test task = 10 minutes (after fix)
- [ ] Test task = 30 minutes (after fix)
- [ ] Test agent crash scenarios
- [ ] Monitor function execution time
- [ ] Verify sandbox lifetime utilization

## Files to Modify

### High Priority
1. `lib/sandbox/agents/claude.ts` - Add timeout to polling loop
2. `lib/sandbox/agents/cursor.ts` - Add timeout to polling loop
3. `lib/sandbox/agents/copilot.ts` - Check for similar pattern

### Medium Priority
4. `app/api/tasks/route.ts` - Implement early exit strategy
5. `lib/sandbox/creation.ts` - Add wrapper script support

### Low Priority
6. `app/api/tasks/[taskId]/continue/route.ts` - Apply same fixes
7. Documentation - Update timeout behavior

## Conclusion

**The sandbox is configured correctly for 30 minutes.** The problem is that the Next.js API route handler (serverless function) has a 5-minute timeout on Vercel Pro plans, and the agent execution code keeps the function alive with an infinite polling loop until this timeout is reached.

**Solution:** Decouple agent execution from serverless function execution time by either:
1. Adding timeouts to prevent infinite loops (immediate)
2. Exiting the function early and monitoring asynchronously (short-term)
3. Using Vercel Background Functions or similar (long-term)
