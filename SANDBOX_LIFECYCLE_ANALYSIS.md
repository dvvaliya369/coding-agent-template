# Sandbox Lifecycle Analysis - Idle Sandboxes Investigation

## Executive Summary

This analysis investigates why sandboxes continue running after the agent process stops and identifies the root causes preventing proper sandbox termination when no active agent is present.

## Key Findings

### 1. Sandbox Lifecycle Design Flaw

**Issue**: Sandboxes are designed to rely on the Vercel Sandbox SDK's automatic timeout mechanism rather than explicit termination when the agent completes.

**Evidence**:
- `lib/sandbox/git.ts:92-94`: Comment explicitly states "Vercel Sandbox automatically shuts down after timeout. No explicit shutdown method available in current SDK"
- `shutdownSandbox()` function only attempts to kill processes with `pkill` but does **not** call any SDK shutdown method
- Recent fixes (commits `5f810db` and `fadbde6`) attempted to address agent timeout issues but did not resolve the core sandbox termination problem

### 2. Conditional Shutdown Logic

**Critical Gap**: Sandbox shutdown is conditional on the `keepAlive` setting, creating scenarios where sandboxes persist unnecessarily.

**Code Locations**:
- `app/api/tasks/route.ts:689-704`: Shutdown only occurs when `keepAlive=false`
- `app/api/tasks/route.ts:730-744`: Error handling also conditionally shuts down based on `keepAlive`
- `app/api/tasks/[taskId]/continue/route.ts:398-414`: Same pattern in continuation endpoint

**Problem**: When `keepAlive=true`, the sandbox persists indefinitely, relying only on the SDK timeout mechanism.

### 3. Missing Explicit Sandbox Stop Call

**Issue**: The `shutdownSandbox()` function does NOT call `sandbox.stop()` which is the proper SDK method to terminate sandboxes.

**Evidence**:
- `lib/sandbox/git.ts:75-100`: The shutdown function only kills processes, doesn't stop the sandbox
- `app/api/tasks/[taskId]/stop-sandbox/route.ts:44`: The manual stop endpoint DOES call `sandbox.stop()` correctly
- `lib/sandbox/sandbox-registry.ts:49`: The killSandbox registry function calls `sandbox.stop()` correctly

**Root Cause**: The main task processing flow uses `shutdownSandbox()` which is incomplete, while manual operations use the correct `sandbox.stop()` method.

### 4. Agent Process Monitoring Issues

**Recent Fixes Applied**:
- Commit `5f810db`: Extended agent wait time from 4 minutes to 10 hours to prevent premature termination
- Commit `fadbde6`: Added sandbox cleanup on agent timeout

**Remaining Issue**: These fixes address agent timeout but don't resolve the core problem: sandboxes continue running after agent completion when `keepAlive=true` is set.

### 5. Timeout Configuration Analysis

**Sandbox Timeout Configuration**:
- `lib/sandbox/creation.ts:73-86`: Sandbox timeout is set during creation: `timeout: timeoutMs`
- Default: 300 minutes (5 hours) from `MAX_SANDBOX_DURATION` environment variable
- `lib/constants.ts:5`: `MAX_SANDBOX_DURATION = parseInt(process.env.MAX_SANDBOX_DURATION || '300', 10)`

**Task Timeout Configuration**:
- `app/api/tasks/route.ts:276`: Task timeout: `TASK_TIMEOUT_MS = maxDuration * 60 * 1000`
- Uses the same `maxDuration` value as sandbox timeout

**Misalignment**: Both task and sandbox use the same timeout value, but:
1. Task timeout triggers first and may leave sandbox running
2. Sandbox timeout is the only cleanup mechanism for `keepAlive=true` scenarios
3. No monitoring ensures sandbox terminates when task completes early

## Root Causes Summary

### Primary Root Cause
**Incomplete Shutdown Implementation**: The `shutdownSandbox()` function in `lib/sandbox/git.ts` does not call `sandbox.stop()`, relying instead on the SDK's automatic timeout mechanism and process killing.

### Secondary Root Causes

1. **Conditional Cleanup Logic**: Sandboxes with `keepAlive=true` are never explicitly terminated, only allowed to timeout naturally after the configured duration (default 5 hours)

2. **No Active Monitoring**: There is no mechanism to detect idle sandboxes (no active agent process) and terminate them proactively

3. **Registry-Only Tracking**: The sandbox registry (`lib/sandbox/sandbox-registry.ts`) is in-memory only, losing track of sandboxes across serverless function invocations

4. **Disconnect Between Task and Sandbox Lifecycle**: Task completion/failure doesn't guarantee sandbox termination when `keepAlive=true`

## Resource Usage Implications

### Current Behavior
- Sandboxes with `keepAlive=false`: Properly terminated (but still missing explicit `stop()` call)
- Sandboxes with `keepAlive=true`: Run for full configured duration (up to 5 hours default) regardless of agent completion
- Idle sandboxes continue consuming:
  - vCPU resources (4 vCPUs per sandbox)
  - Memory
  - Network bandwidth
  - Vercel Sandbox API quota

### Configuration Alignment
Runtime configuration appears correct:
- Timeout properly set during `Sandbox.create()`: `lib/sandbox/creation.ts:86`
- Configured with: `timeout: maxDurationMinutes * 60 * 1000` (milliseconds)
- Default resources: 4 vCPUs, node22 runtime

## Evidence of the Issue

### Code Patterns Showing the Gap

1. **Incomplete shutdown**:
```typescript
// lib/sandbox/git.ts:75-100
export async function shutdownSandbox(sandbox?: Sandbox): Promise<{ success: boolean; error?: string }> {
  try {
    if (sandbox) {
      // Only kills processes, doesn't stop sandbox
      await runCommandInSandbox(sandbox, 'pkill', ['-f', 'node'])
      // ... more pkill commands
    }
    // NOTE: No sandbox.stop() call here!
    return { success: true }
  }
}
```

2. **Correct implementation exists elsewhere**:
```typescript
// app/api/tasks/[taskId]/stop-sandbox/route.ts:44
await sandbox.stop() // ✓ Correct
```

3. **Conditional cleanup prevents termination**:
```typescript
// app/api/tasks/route.ts:689-704
if (keepAlive) {
  await logger.info('Sandbox kept alive for follow-up messages')
  // Sandbox never terminated!
} else {
  unregisterSandbox(taskId)
  await shutdownSandbox(sandbox!) // Uses incomplete shutdown
}
```

## Recommendations

### Immediate Fixes Required

1. **Fix `shutdownSandbox()` Implementation**:
   - Add explicit `sandbox.stop()` call in `lib/sandbox/git.ts`
   - Follow the pattern from `stop-sandbox/route.ts:44`

2. **Implement Idle Detection for KeepAlive Sandboxes**:
   - Add timeout tracking for inactive keepAlive sandboxes
   - Terminate sandboxes that have been idle (no agent activity) for configurable period (e.g., 30 minutes)

3. **Add Sandbox Health Monitoring**:
   - Periodic check for orphaned sandboxes (sandbox exists but task is completed/failed)
   - Background job to cleanup orphaned sandboxes

4. **Persistent Sandbox Tracking**:
   - Move sandbox tracking from in-memory registry to database
   - Add `sandboxCreatedAt`, `lastActivityAt` fields to track idle time

### Long-term Improvements

1. **Separate Task and Sandbox Timeouts**:
   - Task timeout: for agent execution
   - Sandbox idle timeout: for resource cleanup
   - Sandbox maximum lifetime: hard limit

2. **Resource Usage Monitoring**:
   - Track actual sandbox resource consumption
   - Alert on sandboxes exceeding expected lifetime

3. **Graceful Degradation**:
   - When `keepAlive=true`, set a maximum idle period before forced shutdown
   - Warn users before terminating idle keepAlive sandboxes

## Validation Checklist

To validate no idle sandboxes remain:

- [ ] Check sandbox registry for active sandboxes without corresponding running tasks
- [ ] Verify `shutdownSandbox()` actually calls `sandbox.stop()`
- [ ] Confirm sandboxes terminate within expected time after task completion
- [ ] Monitor Vercel Sandbox API for active sandbox count vs expected count
- [ ] Add logging to track sandbox creation vs termination events
- [ ] Implement health check endpoint showing active sandboxes with task status

## Recent Commits Context

The recent fixes addressed agent timeout issues but created new problems:
- Extended agent wait time to 10 hours (commit `5f810db`) prevents premature agent termination
- Added sandbox cleanup on task timeout (commit `fadbde6`)
- However, these changes don't address the core issue: **sandboxes with keepAlive=true never get explicitly terminated**

## Conclusion

The root cause is clear: **The `shutdownSandbox()` function is incomplete and doesn't call `sandbox.stop()`, combined with conditional cleanup logic that allows keepAlive sandboxes to persist indefinitely**. This results in idle sandboxes continuing to consume resources for their full configured lifetime (up to 5 hours by default) even after the agent has completed its work.

The fix requires:
1. Updating `shutdownSandbox()` to call `sandbox.stop()`
2. Implementing idle detection for keepAlive scenarios
3. Adding proper monitoring and cleanup mechanisms

Generated: 2026-02-11
Branch: agent/when-the-sandbox-settings-are-configured-to-run-fo-29-rb-claude
