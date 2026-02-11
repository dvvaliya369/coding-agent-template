# Proposed Code Fixes for Agent Timeout Issue

## Overview

This document provides specific code changes to fix the agent termination issue at the 5-minute mark.

---

## Fix 1: Add Timeout to Polling Loop (Immediate - Low Risk)

### File: `lib/sandbox/agents/claude.ts`

**Location:** Lines 415-430

**Current Code:**
```typescript
// Execute Claude CLI with streaming
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', fullCommand],
  sudo: false,
  detached: true,
  cwd: PROJECT_DIR,
  stdout: captureStdout,
  stderr: captureStderr,
})

await logger.info('Claude command started with output capture, monitoring for completion...')

// Wait for completion - let sandbox timeout handle the hard limit
while (!isCompleted) {
  await new Promise((resolve) => setTimeout(resolve, 1000)) // Wait 1 second
}
```

**Fixed Code:**
```typescript
// Execute Claude CLI with streaming
await sandbox.runCommand({
  cmd: 'sh',
  args: ['-c', fullCommand],
  sudo: false,
  detached: true,
  cwd: PROJECT_DIR,
  stdout: captureStdout,
  stderr: captureStderr,
})

await logger.info('Claude command started with output capture, monitoring for completion...')

// Wait for completion with timeout protection
const MAX_AGENT_WAIT_MS = 25 * 60 * 1000 // 25 minutes (leave 5 min buffer for cleanup)
const agentStartTime = Date.now()
let lastLogTime = Date.now()

while (!isCompleted) {
  const elapsedMs = Date.now() - agentStartTime
  
  // Check if we've exceeded the maximum wait time
  if (elapsedMs > MAX_AGENT_WAIT_MS) {
    await logger.error(`Agent execution timed out after ${Math.floor(elapsedMs / 60000)} minutes`)
    throw new Error(`Agent execution timed out. The agent did not complete within the allocated time.`)
  }
  
  // Log progress every 30 seconds to show we're still alive
  if (Date.now() - lastLogTime > 30000) {
    await logger.info(`Agent still running... (${Math.floor(elapsedMs / 60000)} minutes elapsed)`)
    lastLogTime = Date.now()
  }
  
  await new Promise((resolve) => setTimeout(resolve, 1000)) // Wait 1 second
}

await logger.info('Claude completed successfully')
```

**Benefits:**
- Prevents infinite loops
- Provides progress updates every 30 seconds
- Clear timeout error message
- Leaves 5-minute buffer for cleanup operations

**Apply same fix to:** `lib/sandbox/agents/cursor.ts` (similar pattern at lines 461-481)

---

## Fix 2: Sandbox-Side Wrapper Script (Recommended - Medium Risk)

### File: `lib/sandbox/agents/claude.ts`

**Add new function before `executeClaudeInSandbox`:**

```typescript
/**
 * Creates a wrapper script that runs the agent CLI entirely within the sandbox
 * This decouples agent execution from the serverless function lifetime
 */
async function createAgentWrapperScript(
  sandbox: Sandbox,
  fullCommand: string,
  maxDurationMinutes: number,
  logger: TaskLogger,
): Promise<{ success: boolean; scriptPath: string }> {
  const scriptPath = '/tmp/agent-wrapper.sh'
  const outputLog = '/tmp/agent-output.log'
  const statusFile = '/tmp/agent-status'
  const sessionFile = '/tmp/agent-session'
  
  // Create wrapper script that:
  // 1. Runs agent with timeout
  // 2. Captures output
  // 3. Writes status on completion
  // 4. Extracts session ID if available
  const wrapperScript = `#!/bin/bash
set -e

# Clean up any previous runs
rm -f ${outputLog} ${statusFile} ${sessionFile}

# Initialize status
echo "RUNNING" > ${statusFile}

# Run the agent with timeout
# Use 'timeout' command to enforce hard limit
if timeout ${maxDurationMinutes}m bash -c '${fullCommand.replace(/'/g, "'\\''")}' > ${outputLog} 2>&1; then
  echo "COMPLETED" > ${statusFile}
  
  # Try to extract session ID from output (for resumption)
  if grep -q '"session_id"' ${outputLog}; then
    grep -o '"session_id":"[^"]*"' ${outputLog} | tail -1 | cut -d'"' -f4 > ${sessionFile}
  fi
else
  EXIT_CODE=$?
  if [ $EXIT_CODE -eq 124 ]; then
    echo "TIMEOUT" > ${statusFile}
  else
    echo "FAILED:$EXIT_CODE" > ${statusFile}
  fi
fi

# Ensure status file is written
sync
`

  try {
    // Write the wrapper script to the sandbox
    const writeResult = await runCommandInSandbox(sandbox, 'sh', [
      '-c',
      `cat > ${scriptPath} << 'WRAPPER_EOF'\n${wrapperScript}\nWRAPPER_EOF`,
    ])
    
    if (!writeResult.success) {
      await logger.error('Failed to create wrapper script')
      return { success: false, scriptPath }
    }
    
    // Make it executable
    const chmodResult = await runCommandInSandbox(sandbox, 'chmod', ['+x', scriptPath])
    if (!chmodResult.success) {
      await logger.error('Failed to make wrapper script executable')
      return { success: false, scriptPath }
    }
    
    await logger.info('Agent wrapper script created successfully')
    return { success: true, scriptPath }
  } catch (error) {
    await logger.error(`Error creating wrapper script: ${error}`)
    return { success: false, scriptPath }
  }
}

/**
 * Monitors the agent wrapper script by polling the status file
 * This allows the serverless function to exit while the agent continues
 */
async function monitorAgentExecution(
  sandbox: Sandbox,
  maxDurationMinutes: number,
  logger: TaskLogger,
): Promise<{ success: boolean; sessionId?: string; error?: string }> {
  const statusFile = '/tmp/agent-status'
  const sessionFile = '/tmp/agent-session'
  const outputLog = '/tmp/agent-output.log'
  
  const MAX_WAIT_MS = maxDurationMinutes * 60 * 1000
  const POLL_INTERVAL_MS = 5000 // Check every 5 seconds
  const startTime = Date.now()
  let lastLogTime = Date.now()
  
  while (Date.now() - startTime < MAX_WAIT_MS) {
    // Read status file
    const statusResult = await runCommandInSandbox(sandbox, 'cat', [statusFile])
    
    if (statusResult.success && statusResult.output) {
      const status = statusResult.output.trim()
      
      if (status === 'COMPLETED') {
        await logger.success('Agent completed successfully')
        
        // Try to read session ID
        const sessionResult = await runCommandInSandbox(sandbox, 'cat', [sessionFile])
        const sessionId = sessionResult.success ? sessionResult.output?.trim() : undefined
        
        return { success: true, sessionId }
      } else if (status === 'TIMEOUT') {
        await logger.error('Agent execution timed out')
        return { success: false, error: 'Agent execution timed out' }
      } else if (status.startsWith('FAILED:')) {
        const exitCode = status.split(':')[1]
        await logger.error(`Agent failed with exit code ${exitCode}`)
        
        // Read last 50 lines of output for debugging
        const tailResult = await runCommandInSandbox(sandbox, 'tail', ['-n', '50', outputLog])
        if (tailResult.success && tailResult.output) {
          await logger.error('Agent output (last 50 lines):')
          await logger.error(tailResult.output)
        }
        
        return { success: false, error: `Agent failed with exit code ${exitCode}` }
      }
      // else status === 'RUNNING', continue polling
    }
    
    // Log progress every 30 seconds
    if (Date.now() - lastLogTime > 30000) {
      const elapsedMin = Math.floor((Date.now() - startTime) / 60000)
      await logger.info(`Agent still running... (${elapsedMin} minutes elapsed)`)
      lastLogTime = Date.now()
    }
    
    // Wait before next poll
    await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
  }
  
  // If we get here, we've exceeded max wait time
  await logger.error('Monitoring timed out - agent may still be running in sandbox')
  return { success: false, error: 'Monitoring timeout exceeded' }
}
```

**Modify `executeClaudeInSandbox` function:**

```typescript
export async function executeClaudeInSandbox(
  sandbox: Sandbox,
  instruction: string,
  logger: TaskLogger,
  selectedModel?: string,
  mcpServers?: Connector[],
  isResumed?: boolean,
  sessionId?: string,
  taskId?: string,
  agentMessageId?: string,
  maxDurationMinutes: number = 30, // Add this parameter
): Promise<AgentExecutionResult> {
  let extractedSessionId: string | undefined
  try {
    // ... existing code for CLI installation and setup ...
    
    // Build command with stream-json output format for streaming
    let fullCommand = `${envPrefix} claude --model "${modelToUse}" --dangerously-skip-permissions --output-format stream-json --verbose`
    
    // ... existing code for building command ...
    
    fullCommand += ` "${instruction}"`
    
    await logger.info('Creating agent wrapper script for sandbox-side execution...')
    
    // Create wrapper script
    const wrapperResult = await createAgentWrapperScript(
      sandbox,
      fullCommand,
      maxDurationMinutes,
      logger,
    )
    
    if (!wrapperResult.success) {
      throw new Error('Failed to create agent wrapper script')
    }
    
    // Start the wrapper script in detached mode
    await logger.info('Starting agent in sandbox (detached mode)...')
    await sandbox.runCommand({
      cmd: 'sh',
      args: ['-c', `nohup ${wrapperResult.scriptPath} > /dev/null 2>&1 &`],
      sudo: false,
      detached: true,
      cwd: PROJECT_DIR,
    })
    
    // Monitor execution by polling status file
    await logger.info('Agent started, monitoring execution...')
    const monitorResult = await monitorAgentExecution(
      sandbox,
      maxDurationMinutes,
      logger,
    )
    
    if (!monitorResult.success) {
      return {
        success: false,
        error: monitorResult.error || 'Agent execution failed',
        cliName: 'claude',
        changesDetected: false,
      }
    }
    
    extractedSessionId = monitorResult.sessionId
    
    // Check if any files were modified
    const gitStatusCheck = await runAndLogCommand(sandbox, 'git', ['status', '--porcelain'], logger)
    const hasChanges = gitStatusCheck.success && gitStatusCheck.output?.trim()
    
    return {
      success: true,
      output: `Claude CLI executed successfully${hasChanges ? ' (Changes detected)' : ' (No changes made)'}`,
      cliName: 'claude',
      changesDetected: !!hasChanges,
      sessionId: extractedSessionId,
    }
  } catch (error: unknown) {
    const errorMessage = error instanceof Error ? error.message : 'Failed to execute Claude CLI in sandbox'
    return {
      success: false,
      error: errorMessage,
      cliName: 'claude',
      changesDetected: false,
    }
  }
}
```

**Update function signature in `lib/sandbox/agents/index.ts`:**

```typescript
export async function executeAgentInSandbox(
  sandbox: Sandbox,
  instruction: string,
  agentType: AgentType,
  logger: TaskLogger,
  selectedModel?: string,
  mcpServers?: Connector[],
  onCancellationCheck?: () => Promise<boolean>,
  apiKeys?: {
    OPENAI_API_KEY?: string
    GEMINI_API_KEY?: string
    CURSOR_API_KEY?: string
    ANTHROPIC_API_KEY?: string
    AI_GATEWAY_API_KEY?: string
  },
  isResumed?: boolean,
  sessionId?: string,
  taskId?: string,
  agentMessageId?: string,
  maxDurationMinutes?: number, // Add this parameter
): Promise<AgentExecutionResult> {
  // ... existing code ...
  
  switch (agentType) {
    case 'claude':
      return await executeClaudeInSandbox(
        sandbox,
        instruction,
        logger,
        selectedModel,
        mcpServers,
        isResumed,
        sessionId,
        taskId,
        agentMessageId,
        maxDurationMinutes || 30, // Pass the duration
      )
    // ... other cases ...
  }
}
```

**Update caller in `app/api/tasks/route.ts`:**

```typescript
const agentResult = await executeAgentInSandbox(
  sandbox,
  sanitizedPrompt,
  selectedAgent as AgentType,
  logger,
  selectedModel,
  mcpServers,
  undefined,
  apiKeys,
  undefined, // isResumed
  undefined, // sessionId
  taskId,
  agentMessageId,
  maxDuration, // Pass the max duration from task config
)
```

**Benefits:**
- Agent runs entirely in sandbox (not tied to function lifetime)
- Serverless function can exit quickly (< 1 minute)
- Proper timeout handling with `timeout` command
- Status monitoring via file polling (less resource intensive)
- Session ID extraction for resumption
- Better error handling and logging

---

## Fix 3: Early Function Exit (Alternative - Low Risk)

### File: `app/api/tasks/route.ts`

**Modify `processTask` function to exit early:**

```typescript
async function processTask(
  taskId: string,
  prompt: string,
  repoUrl: string,
  maxDuration: number,
  // ... other params
) {
  let sandbox: Sandbox | null = null
  const logger = createTaskLogger(taskId)
  
  try {
    // ... existing sandbox creation code ...
    
    // Start agent execution
    await logger.info('Starting agent in background...')
    
    // Launch agent asynchronously (don't await)
    executeAgentInBackground(
      sandbox,
      sanitizedPrompt,
      selectedAgent as AgentType,
      logger,
      selectedModel,
      mcpServers,
      apiKeys,
      taskId,
      maxDuration,
    ).catch(async (error) => {
      // Handle errors asynchronously
      console.error('Agent execution failed:', error)
      await logger.error(`Agent execution failed: ${error.message}`)
      await db.update(tasks).set({
        status: 'error',
        error: error.message,
        updatedAt: new Date(),
      }).where(eq(tasks.id, taskId))
    })
    
    // Exit function immediately
    await logger.info('Agent started successfully, running in background')
    await db.update(tasks).set({
      status: 'processing',
      updatedAt: new Date(),
    }).where(eq(tasks.id, taskId))
    
    // Function exits here, agent continues in sandbox
    return
    
  } catch (error) {
    // Handle setup errors
    console.error('Task setup failed:', error)
    await logger.error(`Task setup failed: ${error.message}`)
    throw error
  }
}

async function executeAgentInBackground(
  sandbox: Sandbox,
  prompt: string,
  agentType: AgentType,
  logger: TaskLogger,
  selectedModel?: string,
  mcpServers?: Connector[],
  apiKeys?: any,
  taskId?: string,
  maxDuration?: number,
) {
  // This function runs independently after the HTTP response is sent
  const agentResult = await executeAgentInSandbox(
    sandbox,
    prompt,
    agentType,
    logger,
    selectedModel,
    mcpServers,
    undefined,
    apiKeys,
    undefined,
    undefined,
    taskId,
    generateId(),
    maxDuration,
  )
  
  if (agentResult.success) {
    // Push changes and update task
    await logger.success('Agent completed successfully')
    // ... existing push and cleanup code ...
  } else {
    throw new Error(agentResult.error || 'Agent execution failed')
  }
}
```

**Benefits:**
- Function exits quickly (no timeout issues)
- Agent continues running in sandbox
- Simpler implementation than Fix 2
- Works with existing agent code

**Drawbacks:**
- Still subject to agent CLI timeout issues
- Less control over agent lifecycle
- Harder to debug failures

---

## Recommended Implementation Order

### Phase 1: Immediate (Deploy Today)
1. ✅ Implement **Fix 1** (Add timeout to polling loop)
   - Low risk, high value
   - Prevents infinite loops
   - Provides better error messages
   - Can be deployed immediately

### Phase 2: Short-term (Deploy This Week)
2. ✅ Implement **Fix 2** (Sandbox-side wrapper script)
   - Medium risk, highest value
   - Solves root cause
   - Enables tasks > 5 minutes
   - Requires thorough testing

### Phase 3: Optimization (Optional)
3. ⚠️ Consider **Fix 3** (Early function exit) as alternative
   - Only if Fix 2 is too complex
   - Simpler but less robust
   - Good interim solution

---

## Testing Plan

### Test Case 1: Short Task (< 5 minutes)
- **Expected:** Should work with all fixes
- **Verify:** Task completes successfully
- **Check:** Logs show completion, changes pushed

### Test Case 2: Medium Task (5-10 minutes)
- **Expected:** Fails with current code, succeeds with fixes
- **Verify:** Task runs beyond 5 minutes
- **Check:** No premature termination

### Test Case 3: Long Task (10-30 minutes)
- **Expected:** Succeeds with Fix 2
- **Verify:** Task completes within sandbox lifetime
- **Check:** Proper timeout at 30 minutes if needed

### Test Case 4: Timeout Scenario
- **Expected:** Graceful timeout at configured limit
- **Verify:** Clear error message
- **Check:** Sandbox cleaned up properly

### Test Case 5: Agent Crash
- **Expected:** Detected and reported
- **Verify:** Status file shows FAILED
- **Check:** Error logs captured

---

## Rollback Plan

If issues arise after deployment:

1. **Immediate rollback:** Revert to previous version
2. **Partial rollback:** Keep Fix 1, remove Fix 2
3. **Debug mode:** Add extra logging to identify issues
4. **Gradual rollout:** Deploy to subset of users first

---

## Monitoring

After deployment, monitor:

1. **Task completion rate:** Should increase for tasks > 5 min
2. **Function execution time:** Should decrease significantly
3. **Sandbox utilization:** Should see better usage of 30-min window
4. **Error rates:** Should decrease for timeout errors
5. **User feedback:** Check for reports of hanging tasks

---

## Conclusion

**Recommended approach:** Implement Fix 1 immediately, then Fix 2 within the week.

This provides:
- Immediate improvement (Fix 1)
- Long-term solution (Fix 2)
- Minimal risk
- Clear upgrade path
