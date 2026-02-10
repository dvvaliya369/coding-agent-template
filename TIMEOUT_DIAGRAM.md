# Agent Timeout Issue - Visual Diagram

## Current Architecture (Problematic)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Serverless Function                       │
│                         (5-minute timeout on Pro)                        │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Next.js API Route Handler                                        │   │
│  │ (app/api/tasks/route.ts)                                         │   │
│  │                                                                   │   │
│  │  1. Create Sandbox (30-min lifetime) ✅                          │   │
│  │     └─> Sandbox.create({ timeout: 30 * 60 * 1000 })             │   │
│  │                                                                   │   │
│  │  2. Execute Agent CLI (detached) ⚠️                              │   │
│  │     └─> sandbox.runCommand({ detached: true })                   │   │
│  │                                                                   │   │
│  │  3. Enter Infinite Polling Loop ❌                               │   │
│  │     ┌──────────────────────────────────────┐                     │   │
│  │     │ while (!isCompleted) {               │                     │   │
│  │     │   await sleep(1000)  // No timeout!  │                     │   │
│  │     │ }                                    │                     │   │
│  │     └──────────────────────────────────────┘                     │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ⏰ Time: 0 ──────────────────────> 5 minutes ──────> KILLED ❌         │
└─────────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ SIGKILL
                                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Sandbox                                   │
│                         (30-minute lifetime)                             │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Agent CLI Process (claude, cursor, etc.)                         │   │
│  │                                                                   │   │
│  │  Running... ✅                                                   │   │
│  │  Making changes... ✅                                            │   │
│  │  Still working... ✅                                             │   │
│  │  Almost done... ⚠️                                               │   │
│  │  TERMINATED (parent killed) ❌                                   │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ⏰ Time: 0 ──────────────────────> 5 min ──> 30 min (unused)           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Timeline Breakdown

```
Time    Serverless Function              Sandbox                Agent Process
────────────────────────────────────────────────────────────────────────────
0:00    │ Start                          │ Created              │
        │ Create sandbox ──────────────> │                      │
        │                                │                      │
0:30    │ Start agent ───────────────────────────────────────> │ Started
        │ Enter polling loop             │                      │ Working...
        │ while (!isCompleted)...        │                      │
        │                                │                      │
1:00    │ Still polling...               │ Alive                │ Working...
        │                                │                      │
2:00    │ Still polling...               │ Alive                │ Working...
        │                                │                      │
3:00    │ Still polling...               │ Alive                │ Working...
        │                                │                      │
4:00    │ Still polling...               │ Alive                │ Working...
        │                                │                      │
4:59    │ Still polling...               │ Alive                │ Almost done
        │                                │                      │
5:00    │ ❌ TIMEOUT KILLED              │ Alive (orphaned)     │ ❌ KILLED
        │                                │                      │
5:01    │ (dead)                         │ Alive                │ (dead)
        │                                │                      │
...     │                                │                      │
30:00   │                                │ ❌ Expires           │
```

## Problem Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                    Timeout Mismatch                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Serverless Function Timeout (Vercel Pro):                      │
│  ├─────────────────────────────────────┤                        │
│  0                                   5 min                       │
│                                                                  │
│  Sandbox Lifetime (Configured):                                 │
│  ├──────────────────────────────────────────────────────────────┤
│  0                                                          30 min│
│                                                                  │
│  Agent Execution Time (Actual):                                 │
│  ├────────────────────────────────────────────────┤             │
│  0                                              10 min           │
│                                                                  │
│  Result:                                                         │
│  ├─────────────────────────────────────┤ ❌ KILLED              │
│  0                                   5 min                       │
│                                      ▲                           │
│                                      │                           │
│                              Function timeout kills              │
│                              agent before completion             │
└─────────────────────────────────────────────────────────────────┘
```

## Code Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│ User creates task                                                     │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│ POST /api/tasks                                                       │
│ ├─ Validate input                                                     │
│ ├─ Create task in DB                                                  │
│ ├─ Return response to user ✅                                         │
│ └─ after(() => processTaskWithTimeout(...))  ⚠️                      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│ processTaskWithTimeout()                                              │
│ ├─ Set timeout: maxDuration * 60 * 1000 (30 min)                     │
│ └─ Promise.race([processTask(), timeoutPromise])                     │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│ processTask()                                                         │
│ ├─ createSandbox() → Sandbox with 30-min lifetime ✅                 │
│ └─ executeAgentInSandbox()                                            │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│ executeAgentInSandbox() (lib/sandbox/agents/claude.ts)               │
│ ├─ Install agent CLI                                                  │
│ ├─ Build command string                                               │
│ ├─ sandbox.runCommand({ detached: true }) ⚠️                         │
│ └─ while (!isCompleted) { await sleep(1000) } ❌ INFINITE LOOP       │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
                    ⏰ 5 minutes pass
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Vercel Platform                                                       │
│ ├─ Serverless function timeout reached (5 min)                       │
│ ├─ Send SIGKILL to function process                                  │
│ ├─ Terminate all child processes                                     │
│ └─ Agent CLI process killed ❌                                        │
└──────────────────────────────────────────────────────────────────────┘
```

## Comparison: Expected vs. Actual

### Expected Behavior
```
┌─────────────────────────────────────────────────────────────────┐
│ Task Execution                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ 1. Create sandbox (30 min) ✅                                   │
│ 2. Start agent ✅                                               │
│ 3. Agent works for 10 minutes ✅                                │
│ 4. Agent completes ✅                                           │
│ 5. Push changes ✅                                              │
│ 6. Shutdown sandbox ✅                                          │
│                                                                  │
│ Total time: ~10 minutes                                         │
│ Status: Success ✅                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Actual Behavior
```
┌─────────────────────────────────────────────────────────────────┐
│ Task Execution                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ 1. Create sandbox (30 min) ✅                                   │
│ 2. Start agent ✅                                               │
│ 3. Agent works for 5 minutes ⚠️                                 │
│ 4. Serverless function timeout ❌                               │
│ 5. Agent killed mid-execution ❌                                │
│ 6. Changes lost ❌                                              │
│ 7. Sandbox orphaned (still alive) ⚠️                            │
│                                                                  │
│ Total time: 5 minutes (forced)                                  │
│ Status: Failed ❌                                               │
└─────────────────────────────────────────────────────────────────┘
```

## Solution Architecture

### Recommended: Sandbox-Side Execution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Serverless Function                       │
│                         (5-minute timeout on Pro)                        │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Next.js API Route Handler                                        │   │
│  │                                                                   │   │
│  │  1. Create Sandbox (30-min lifetime) ✅                          │   │
│  │  2. Create wrapper script in sandbox ✅                          │   │
│  │  3. Start wrapper script (detached) ✅                           │   │
│  │  4. Exit function immediately ✅                                 │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ⏰ Time: 0 ──> 30 seconds ──> EXITS CLEANLY ✅                         │
└─────────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ No dependency
                                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Vercel Sandbox                                   │
│                         (30-minute lifetime)                             │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Wrapper Script (runs independently)                              │   │
│  │                                                                   │   │
│  │  #!/bin/bash                                                      │   │
│  │  timeout 30m claude "..." > /tmp/agent.log 2>&1                  │   │
│  │  echo "COMPLETED" > /tmp/status                                  │   │
│  │  git add . && git commit -m "..." && git push                    │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ⏰ Time: 0 ──────────────────────> 10 min ──> 30 min ✅                │
│                                                                           │
│  Monitoring: Separate polling endpoint checks /tmp/status every 10s      │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Differences

| Aspect | Current (Broken) | Proposed (Fixed) |
|--------|------------------|------------------|
| **Agent execution** | In serverless function | In sandbox |
| **Function lifetime** | 5 minutes (timeout) | 30 seconds (quick exit) |
| **Agent lifetime** | Tied to function | Independent (30 min) |
| **Monitoring** | Blocking while loop | Async polling endpoint |
| **Timeout handling** | None (infinite loop) | `timeout` command |
| **Failure mode** | Hard kill at 5 min | Graceful timeout at 30 min |
| **Resource usage** | Function + Sandbox | Sandbox only |
| **Cost** | Higher (long function) | Lower (short function) |

## Summary

The issue is a **timeout mismatch** between:
- **Serverless function timeout:** 5 minutes (Vercel Pro)
- **Sandbox lifetime:** 30 minutes (configured)
- **Agent execution time:** Variable (can exceed 5 minutes)

The infinite polling loop keeps the serverless function alive until the platform kills it at 5 minutes, terminating the agent process prematurely.

**Solution:** Decouple agent execution from serverless function execution by running the agent entirely within the sandbox and monitoring asynchronously.
