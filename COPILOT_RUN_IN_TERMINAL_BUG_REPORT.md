# GitHub Copilot Bug Report: `run_in_terminal` Tool Fails with Network Error

## Summary
The `run_in_terminal` tool consistently fails after user approval in both VS Code Stable and Insiders. Terminal commands execute successfully, but the tool call never completes, leaving the chat interface in an indefinite "executing" state. Console logs reveal network/API communication failures (`undici` fetch termination).

## Environment
- **OS**: Windows 10/11 (Build 10.0.26200)
- **VS Code Version**: 1.106.3 (Stable, user setup)
  - Commit: bf9252a2fb45be6893dd8870c0bf37e2e1766d61
  - Date: 2025-11-25T22:28:18.024Z
  - Electron: 37.7.0
  - Chromium: 138.0.7204.251
  - Node.js: 22.20.0
- **Also Tested**: VS Code Insiders (build a44beb9c39) - same issue
- **GitHub Copilot**: 1.388.0
- **Network**: No VPN, Windows Firewall active
- **Other Copilot Features**: ✅ Working normally (chat, code completion)

## Issue Description

### What Happens
1. Copilot requests to run a terminal command via `run_in_terminal` tool
2. User sees approval dialog with "Allow" and "Skip" buttons
3. User clicks "Allow"
4. **Terminal command executes successfully** ✅
5. **Tool call never completes** ❌
   - Chat interface shows persistent spinner
   - Stop button remains active
   - Copilot never receives completion confirmation
   - Workflow is blocked indefinitely

### Expected Behavior
- Tool call should complete after command execution
- Copilot should receive command output
- Chat interface should return to ready state
- User can continue conversation

### Actual Behavior
- Command runs in terminal successfully
- Tool call hangs indefinitely in chat
- Copilot appears "stuck" waiting for response
- User must manually stop the operation

## Reproduction Steps
1. Open VS Code with GitHub Copilot enabled
2. Open a chat with Copilot
3. Request Copilot to run any terminal command (e.g., "run echo test in the terminal")
4. Click "Allow" on the approval dialog
5. Observe:
   - Terminal executes command successfully
   - Chat interface remains in "executing" state with spinner
   - Tool call never completes

**Reproduces**: 100% of the time across multiple commands and shell types (cmd, PowerShell, bash)

## Console Error Logs

### Error Sequence (from Developer Tools Console)

**1. Before "Allow" button click:**
```
[Extension Host] error parsing json: SyntaxError: Unexpected token 'N', "No findings found." is not valid JSON
```

**2. Immediately after clicking "Allow":**
```
ERR [LanguageModelToolsService#invokeTool] Error from tool run_in_terminal with parameters 
{"command":"echo \"Test at 16:40 - December 8, 2025\"","explanation":"Testing...","isBackground":false}:
Canceled: Canceled: Canceled
    at Hwt.execute (vscode-file://vscode-app/.../workbench.desktop.main.js:2860:1141)
    at async noe.invoke (vscode-file://vscode-app/.../workbench.desktop.main.js:2946:1509)
    at async Cdt.invokeTool (vscode-file://vscode-app/.../workbench.desktop.main.js:2239:4834)
    at async ZKt.$invokeTool (vscode-file://vscode-app/.../workbench.desktop.main.js:511:13328)
```

**3. ~1 second later (unhandled promise rejection):**
```
[Extension Host] rejected promise not handled within 1 second: TypeError: terminated

stack trace: TypeError: terminated
	at Fetch.onAborted (node:internal/deps/undici/undici:11132:53)
	at Fetch.terminate (node:internal/deps/undici/undici:10290:14)
	at Object.onError (node:internal/deps/undici/undici:11253:38)
	at Socket.onHttpSocketClose (...undici/lib/dispatcher/client-h1.js:939:10)
	at TCP.<anonymous> (node:net:346:12)
```

**4. Final error:**
```
ERR An unknown error occurred. Please consult the log for more details.
```

## Analysis

### Key Findings
- **Terminal execution works**: Commands run successfully, output is visible
- **Callback mechanism fails**: Tool completion status never reaches Copilot
- **Network error**: `undici` HTTP client aborts fetch request
- **TCP socket closes**: Network connection failure during tool execution
- **Triple cancellation**: Internal cancellation cascade in tool execution pipeline

### Root Cause Hypothesis
The `run_in_terminal` tool attempts to communicate with GitHub Copilot's backend API (likely to stream results or report completion status), but this HTTP request fails with a network termination error. The terminal command itself executes correctly, but the async tool invocation never resolves because the callback to Copilot's service fails.

This suggests:
- Potential issue with Copilot service endpoint
- VS Code's `undici` HTTP client having problems with Copilot's API
- Network communication failure between Extension Host and Copilot backend
- Possible timeout or connection handling bug

## Testing Performed

### Commands Tested (All Failed)
- `echo "test"` (cmd)
- `echo "Testing run_in_terminal in VS Code Stable"` (cmd)
- `echo "Final test with clean prompt - December 9, 2025"` (PowerShell with minimal prompt)
- `go version` (PowerShell)
- `Remove-Item` (PowerShell)

### Environments Tested
- ✅ VS Code Stable 1.106.3 - **Issue present**
- ✅ VS Code Insiders (build a44beb9c39) - **Issue present**
- ✅ Multiple shell types (cmd, PowerShell, bash) - **All affected**
- ✅ Fresh terminal sessions - **No difference**
- ✅ **Clean prompt without RPROMPT** - **Issue persists** (tested December 9, 2025)

### RPROMPT Investigation
Based on https://github.com/orgs/community/discussions/161238, tested whether right-aligned prompts (RPROMPT) were causing terminal output parsing issues:

**Actions Taken:**
- Fixed Oh My Posh v3 configuration in PowerShell profiles
- Added conditional to skip Oh My Posh in PowerShell Extension terminals  
- Created `Microsoft.VSCode_profile.ps1` with simple default prompt
- Verified complete removal of RPROMPT from all terminal types
- Tested with minimal prompt: `PS D:\Users\{username}\dev\slate>`

**Result:** Issue persists with identical errors even with completely clean prompt. RPROMPT is **definitively NOT the root cause**.

**Note:** The GitHub discussion at https://github.com/orgs/community/discussions/161238#discussioncomment-15100771 suggests similar network/API issues affecting terminal tool execution, consistent with our findings.

### What Works
- ✅ GitHub Copilot chat responses
- ✅ Code completions
- ✅ All other Copilot features
- ✅ Terminal commands (when run manually)
- ❌ Only `run_in_terminal` tool is broken

## Historical Context
- User reports `run_in_terminal` worked in VS Code Insiders as recently as 3 weeks ago
- Issue appears to be a recent regression
- Affects both Stable and Insiders builds currently
- Similar issues reported by other users: https://github.com/orgs/community/discussions/161238

## Impact
- **Severity**: Medium - Workaround available (manual command execution)
- **Frequency**: 100% reproduction rate
- **User Impact**: Disrupts AI-assisted workflow, requires manual intervention
- **Affected Feature**: `run_in_terminal` tool only (other Copilot features work)

## Workaround
Copilot can provide commands for users to execute manually. Users copy the command, run it in terminal, and share output if needed.

## Additional Information
- No VPN in use
- Windows Firewall active (standard configuration)
- No proxy configuration
- Network connectivity to GitHub services appears normal (other Copilot features work)
- Issue persists across VS Code restarts
- Issue persists across different workspace folders

## Request
Please investigate the network communication failure in the `run_in_terminal` tool's callback mechanism. The `undici` fetch termination and TCP socket closure suggest a backend API communication issue rather than a local VS Code problem.

---

**Full diagnostic log available at**: COPILOT_RUN_IN_TERMINAL_DIAGNOSTICS.md
