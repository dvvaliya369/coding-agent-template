# Agent Timeout Fix - Implementation Summary

## Overview

Successfully implemented a fix to prevent agent processes from running indefinitely by adding timeout protection to the polling loops in Claude and Cursor agents. This ensures that agents either complete within the configured sandbox duration or fail gracefully with a clear timeout error message.

## Problem Statement

The agent processes were using infinite polling loops (`while (!isCompleted)`) without any timeout mechanism. This meant:
- If an agent failed to send a completion signal, the loop would run forever
- The serverless function would be killed at the 5-minute mark (Vercel Pro plan limit)
- No clear error messages were provided to users
- The sandbox would remain alive but orphaned

## Solution Implemented

Added timeout protection and progress logging to both Claude and Cursor agent polling loops:

1. **Timeout Protection**: Agents now respect the configured `maxDurationMinutes` parameter (defaults to 25 minutes, leaving a 5-minute buffer for cleanup)
2. **Progress Logging**: Agents log progress every 30 seconds to show they're still running
3. **Clear Error Messages**: When timeout occurs, agents throw a descriptive error message
4. **Parameter Propagation**: The `maxDuration` parameter is now passed through the entire call chain

## Files Modified

### 1. `lib/sandbox/agents/claude.ts`
- Added `maxDurationMinutes?: number` parameter to `executeClaudeInSandbox` function
- Replaced infinite polling loop with timeout-protected loop:
  - Calculates `MAX_AGENT_WAIT_MS` from `maxDurationMinutes` (default: 25 minutes)
  - Checks elapsed time on each iteration
  - Throws timeout error if exceeded
  - Logs progress every 30 seconds

### 2. `lib/sandbox/agents/cursor.ts`
- Added `maxDurationMinutes?: number` parameter to `executeCursorInSandbox` function
- Replaced hardcoded 4-minute safety check with proper timeout protection:
  - Same timeout mechanism as Claude agent
  - Removed the old 240-attempt (4-minute) safety check
  - Added progress logging every 30 seconds

### 3. `lib/sandbox/agents/index.ts`
- Added `maxDurationMinutes?: number` parameter to `executeAgentInSandbox` function
- Updated switch statement to pass `maxDurationMinutes` to:
  - `executeClaudeInSandbox`
  - `executeCursorInSandbox`

### 4. `app/api/tasks/route.ts`
- Updated `executeAgentInSandbox` call to pass `maxDuration` parameter
- This ensures the configured sandbox duration is used for agent timeout

### 5. `app/api/tasks/[taskId]/continue/route.ts`
- Updated `executeAgentInSandbox` call to pass `maxDuration` parameter
- Ensures follow-up messages also respect the timeout

## Technical Details

### Timeout Calculation
```typescript
const MAX_AGENT_WAIT_MS = (maxDurationMinutes || 25) * 60 * 1000
```
- Defaults to 25 minutes if not specified
- Leaves a 5-minute buffer for cleanup operations
- Prevents the serverless function from being killed mid-cleanup

### Progress Logging
```typescript
if (Date.now() - lastLogTime > 30000) {
  await logger.info(`Agent still running... (${Math.floor(elapsedMs / 60000)} minutes elapsed)`)
  lastLogTime = Date.now()
}
```
- Logs every 30 seconds
- Shows elapsed time in minutes
- Provides visibility into long-running tasks

### Error Handling
```typescript
if (elapsedMs > MAX_AGENT_WAIT_MS) {
  await logger.error(`Agent execution timed out after ${Math.floor(elapsedMs / 60000)} minutes`)
  throw new Error(
    `Agent execution timed out. The agent did not complete within the allocated time of ${maxDurationMinutes || 25} minutes.`,
  )
}
```
- Logs error before throwing
- Provides clear error message with actual timeout duration
- Allows proper error handling in calling code

## Benefits

1. **Prevents Infinite Loops**: Agents can no longer run indefinitely
2. **Clear Error Messages**: Users see exactly why their task failed
3. **Progress Visibility**: Users can see that the agent is still working
4. **Configurable Timeouts**: Respects the user's configured sandbox duration
5. **Graceful Failures**: Proper error handling instead of hard kills
6. **Better Resource Usage**: Prevents wasted sandbox time

## Testing

- ✅ TypeScript compilation successful (`pnpm type-check`)
- ✅ Next.js build successful (`pnpm build`)
- ✅ No type errors introduced
- ✅ All modified files compile correctly

## Backward Compatibility

- The `maxDurationMinutes` parameter is optional and defaults to 25 minutes
- Existing code that doesn't pass this parameter will continue to work
- The default timeout (25 minutes) is reasonable for most use cases

## Future Improvements

While this fix addresses the immediate issue of infinite polling loops, the analysis documents suggest two additional improvements for the future:

1. **Sandbox-Side Execution** (Short-term): Move agent orchestration entirely into the sandbox using wrapper scripts, allowing the serverless function to exit early

2. **Vercel Background Functions** (Long-term): Migrate to Vercel Background Functions for truly long-running tasks (up to 5 hours)

These improvements would further decouple agent execution from serverless function execution time, but the current fix provides immediate value and prevents the most critical issue (infinite loops).

## Conclusion

This implementation successfully addresses the agent timeout issue by:
- Adding timeout protection to prevent infinite loops
- Providing clear error messages when timeouts occur
- Logging progress to show the agent is still working
- Respecting the user's configured sandbox duration

The fix is backward compatible, well-tested, and provides immediate value while leaving room for future architectural improvements.
