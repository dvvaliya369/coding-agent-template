# Analysis: Agent Process Termination at 5-Minute Mark

## Executive Summary

The agent process is being terminated around the 5-minute mark **NOT** due to sandbox runtime configuration issues, but due to **detached process execution combined with an infinite polling loop** in the agent execution code. The sandbox runtime is correctly configured for 30 minutes, but the agent CLI commands are executed as detached processes without proper timeout handling.

---

## Root Cause Analysis

### 1. **Detached Process Execution Without Timeout**

**Location:** `lib/sandbox/agents/claude.ts` (lines 415-430)

```typescript
// Execute Claude CLI with streaming
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', fullCommand],
  sudo: false,
  detached: true,  // ⚠️ CRITICAL: Process runs in background
  cwd: PROJECT_DIR,
  stdout: captureStdout,
  stderr: captureStderr,
})

await logger.info('Claude command started with output capture, monitoring for completion...')

// Wait for completion - let sandbox timeout handle the hard limit
while (!isCompleted) {  // ⚠️ INFINITE LOOP waiting for completion signal
  await new Promise((resolve) => setTimeout(resolve, 1000)) // Wait 1 second
}
```

**The Problem:**
- The agent CLI command is executed with `detached: true`, meaning it runs as a background process
- The code then enters an **infinite `while` loop** waiting for `isCompleted` to become `true`
- `isCompleted` is set to `true` only when the streaming output parser detects a `result` chunk with `type: 'result'`
- **If the agent CLI process crashes, hangs, or fails to send the completion signal, the loop runs forever**

### 2. **No Timeout on the Detached Process**

The `sandbox.runCommand()` call with `detached: true` does **NOT** respect any timeout configuration. The detached process can run indefinitely until:
- The sandbox itself expires (30 minutes in this case)
- The process is manually killed
- The process completes naturally

However, the **polling loop** (`while (!isCompleted)`) has no timeout mechanism, so if the agent process:
- Crashes silently
- Hangs on I/O
- Fails to send the completion signal
- Gets killed by an external watchdog

The loop will continue polling forever, or until some external timeout kills the entire execution context.

### 3. **Likely Culprit: Server-Side Execution Timeout**

**Evidence from the code:**

In `app/api/tasks/route.ts` (lines 256-280), there's a `processTaskWithTimeout` function:

```typescript
async function processTaskWithTimeout(
  taskId: string,
  prompt: string,
  repoUrl: string,
  maxDuration: number,  // This is the sandbox lifetime (30 minutes)
  // ... other params
) {
  const TASK_TIMEOUT_MS = maxDuration * 60 * 1000 // Convert minutes to milliseconds

  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      reject(new Error(`Task execution timed out after ${maxDuration} minutes`))
    }, TASK_TIMEOUT_MS)
  })

  try {
    await Promise.race([
      processTask(...),
      timeoutPromise,
    ])
  } catch (error: unknown) {
    // Handle timeout
  }
}
```

**However**, this timeout is for the **entire task execution**, not the agent CLI process itself. The issue is that:

1. **Next.js 15 `after()` function** (used in `app/api/tasks/route.ts` line 220) allows background execution after the HTTP response is sent
2. **Vercel serverless functions** have a maximum execution time limit (typically **5 minutes** for Hobby/Pro plans, 15 minutes for Enterprise)
3. Even though the code uses `after()` to continue execution after the response, **the serverless function execution context is still subject to platform limits**

### 4. **The 5-Minute Connection**

**Vercel Serverless Function Limits:**
- **Hobby Plan:** 10 seconds
- **Pro Plan:** 5 minutes (300 seconds) ⚠️ **THIS IS THE CULPRIT**
- **Enterprise Plan:** 15 minutes (900 seconds)

The agent process is being killed at the **5-minute mark** because:
1. The Next.js API route handler runs in a Vercel serverless function
2. Even with `after()`, the function execution context has a hard limit
3. When the serverless function reaches its 5-minute timeout, **all processes are terminated**, including:
   - The detached agent CLI process
   - The polling loop waiting for completion
   - Any background streams

---

## Evidence Supporting This Analysis

### 1. **Detached Process Pattern in Multiple Agents**

The same pattern exists in:
- `lib/sandbox/agents/claude.ts` (line 419)
- `lib/sandbox/agents/cursor.ts` (line 468)

Both use `detached: true` with infinite polling loops.

### 2. **No Explicit Timeout on Agent Execution**

In `lib/sandbox/agents/index.ts`, the `executeAgentInSandbox` function has **no timeout parameter** and relies entirely on external timeouts.

### 3. **Sandbox Timeout vs. Function Timeout Mismatch**

- **Sandbox timeout:** 30 minutes (configured correctly in `lib/sandbox/creation.ts` line 77)
- **Function timeout:** 5 minutes (Vercel Pro plan limit)
- **Result:** Function times out before sandbox expires

---

## Why This Happens at ~5 Minutes

The termination occurs at approximately 5 minutes because:

1. **Vercel Pro Plan Limit:** Serverless functions on Vercel Pro have a 5-minute (300-second) execution limit
2. **After() Limitation:** While Next.js 15's `after()` allows background work after the HTTP response, it **does not extend the serverless function execution time**
3. **Hard Kill:** When the function reaches 5 minutes, Vercel's infrastructure performs a hard kill of the execution context
4. **Process Termination:** All child processes (including the detached agent CLI) are terminated via SIGKILL or similar

---

## Solutions

### Solution 1: **Move Agent Execution to Sandbox-Side Script** (Recommended)

Instead of running the agent CLI as a detached process from the Next.js function, create a wrapper script that runs **inside the sandbox** and manages the agent lifecycle:

```typescript
// Create a wrapper script in the sandbox
const wrapperScript = `
#!/bin/bash
set -e

# Run the agent CLI with timeout
timeout ${maxDuration}m ${fullCommand} > /tmp/agent-output.log 2>&1

# Signal completion
echo "AGENT_COMPLETED" > /tmp/agent-status
`

// Execute the wrapper script
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', wrapperScript],
  detached: true,
  cwd: PROJECT_DIR,
})

// Poll for completion by checking the status file
const startTime = Date.now()
const maxWaitMs = maxDuration * 60 * 1000

while (Date.now() - startTime < maxWaitMs) {
  const statusCheck = await sandbox.runCommand('cat', ['/tmp/agent-status'])
  if (statusCheck.output?.includes('AGENT_COMPLETED')) {
    break
  }
  await new Promise(resolve => setTimeout(resolve, 5000)) // Check every 5 seconds
}
```

**Benefits:**
- Agent runs entirely within the sandbox (not tied to function execution time)
- Proper timeout handling with `timeout` command
- Polling is less frequent (every 5 seconds instead of every 1 second)
- Function can exit early, letting the sandbox continue

### Solution 2: **Add Timeout to Polling Loop**

Add a maximum wait time to the infinite polling loop:

```typescript
const MAX_AGENT_WAIT_MS = maxDuration * 60 * 1000
const agentStartTime = Date.now()

while (!isCompleted) {
  if (Date.now() - agentStartTime > MAX_AGENT_WAIT_MS) {
    throw new Error(`Agent execution timed out after ${maxDuration} minutes`)
  }
  await new Promise((resolve) => setTimeout(resolve, 1000))
}
```

**Benefits:**
- Prevents infinite loops
- Provides clear timeout error messages

**Limitations:**
- Still subject to serverless function timeout (5 minutes)
- Doesn't solve the root cause

### Solution 3: **Use Vercel Background Functions** (Best Long-Term Solution)

Migrate agent execution to Vercel Background Functions (if available on your plan):

```typescript
// app/api/tasks/route.ts
import { background } from '@vercel/functions'

export const POST = background(async (req) => {
  // This function can run for up to 5 hours
  await processTask(...)
})
```

**Benefits:**
- Execution time up to 5 hours (depending on plan)
- Designed for long-running tasks
- No serverless function timeout issues

**Limitations:**
- Requires Vercel Enterprise plan or specific add-on
- Different pricing model

### Solution 4: **Implement Heartbeat Mechanism**

Instead of waiting for completion, implement a heartbeat system:

```typescript
// In the Next.js function
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', fullCommand],
  detached: true,
  cwd: PROJECT_DIR,
})

// Exit the function immediately
await logger.info('Agent started, monitoring via heartbeat')
return

// Separate monitoring endpoint checks sandbox status
// GET /api/tasks/[taskId]/status
// - Checks if agent process is still running
// - Reads output files from sandbox
// - Updates task status in database
```

**Benefits:**
- Function exits quickly (no timeout issues)
- Monitoring happens via separate, lightweight requests
- Agent runs independently in sandbox

---

## Recommended Implementation Plan

### Phase 1: Immediate Fix (Solution 2)
1. Add timeout to polling loops in `claude.ts` and `cursor.ts`
2. Add better error handling for agent process failures
3. Deploy and test

### Phase 2: Architectural Improvement (Solution 1)
1. Create sandbox-side wrapper scripts for each agent
2. Implement file-based status checking
3. Reduce polling frequency
4. Test with long-running tasks (>5 minutes)

### Phase 3: Long-Term Solution (Solution 3 or 4)
1. Evaluate Vercel Background Functions for your use case
2. If not available, implement heartbeat monitoring system
3. Migrate agent execution to be fully asynchronous

---

## Testing Recommendations

1. **Test with 10-minute task:** Verify agent can run beyond 5 minutes
2. **Test with 30-minute task:** Verify full sandbox lifetime is utilized
3. **Test agent crash scenarios:** Ensure timeout handling works correctly
4. **Monitor serverless function execution time:** Confirm function exits before 5-minute limit

---

## Conclusion

The agent process termination at ~5 minutes is caused by **Vercel serverless function execution limits**, not sandbox configuration issues. The sandbox is correctly configured for 30 minutes, but the Next.js API route handler (which orchestrates the agent execution) is subject to a 5-minute timeout on Vercel Pro plans.

The infinite polling loop in the agent execution code (`while (!isCompleted)`) exacerbates this issue by keeping the function alive until the hard timeout is reached, at which point all processes are killed.

**Immediate action required:** Implement Solution 2 (add timeout to polling loop) to prevent infinite loops and provide better error messages.

**Long-term action required:** Implement Solution 1 or 4 to decouple agent execution from serverless function execution time.
