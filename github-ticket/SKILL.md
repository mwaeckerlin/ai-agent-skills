---
name: github-ticket
description: Automated GitHub issue management with Copilot - create issues, assign to Copilot, continuously monitor until complete, and send Telegram notification only when finished
---

# GitHub Ticket Management

Automated GitHub issue management with Copilot integration.

## Overview

When the user requests an issue creation in a specific project:

1. **Create GitHub Issue** with @copilot mention
2. **Assign Copilot** as assignee (MANDATORY!)
3. **Start Monitoring** - every 10 minutes until complete
4. **Review Implementation** when Copilot completes
5. **Provide Feedback** if improvements needed
6. **SEND TELEGRAM MESSAGE** - ONLY WHEN COMPLETE!

## ABSOLUTELY MANDATORY RULES

### 🚨 RULE #1: CONTINUOUS MONITORING

**Set a new 10-minute timer every time, never stop!**

**AS LONG AS Copilot is NOT finished:**
- CREATE NEW 10-MINUTE TIMER (`at` scheduling)
- REPEAT UNTIL COPILOT FINISHED!
- NEVER STOP UNTIL COMPLETE!

### 🚨 RULE #2: TELEGRAM MESSAGE - ONLY WHEN FINISHED

**Send a Telegram message ONLY upon COMPLETION!**

**DURING MONITORING:**
- NO Telegram message during intermediate checks
- Only silent check and continue monitoring

**WHEN FINISHED:**
- ✅ Copilot has completed all work
- ✅ Issue closed or PR merged
- ✅ Implementation is reviewed
- 📱 NOW send TELEGRAM MESSAGE!

## Prerequisites

Required tools and capabilities:
- `mcp-github` skill for GitHub operations
- `openclaw-mcp-gateway` for cron job management
- Telegram integration (must be configured and operational)

## Workflow

### Step 1: Create Issue

Use `github_issues_rest`:

```
operationId: "issues/create"
parameters:
  owner: "repo-owner"
  repo: "repo-name"
  title: "[Auto-Generated] <Issue Title>"
  body: "@copilot \n\n<Detailed description>\n\n<Requirements and acceptance criteria>"
  labels: ["automated", "copilot"]
```

### Step 2: Assign Copilot (MANDATORY!)

**IMPORTANT: @copilot mention alone is NOT enough! Must assign Copilot:**

```
operationId: "issues/add-assignees"
parameters:
  owner: "repo-owner"
  repo: "repo-name"
  issue_number: <issue-id>
  assignees: ["copilot"]
```

### Step 3: Start Monitoring (SILENT!)

**FIRST 10-MINUTE TIMER:**

```
name: "copilot-monitor-<issue-id>-1" # Unique name
schedule:
  kind: "at"
  at: "<current_datetime + 10 minutes>"  # 10 minutes from now
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Check Copilot status for issue #<issue-id>. Perform a silent status check — do not send any Telegram notification."
delivery:
  mode: "none"  # NO Telegram message!
```

### Step 4: MONITORING LOGIC (REPEAT!)

**IN EVERY MONITORING CHECK:**

1. **Check Copilot status:**
   - Is issue closed?
   - Are there pull requests?
   - Are there commits?

2. **If NOT FINISHED:**
   - CREATE NEW 10-MINUTE TIMER
   - Use unique names (-2, -3, etc.)
   - NEVER STOP!

3. **If FINISHED:**
   - Review implementation
   - SEND TELEGRAM MESSAGE

### Step 5: Implementation Review (When Finished)

1. **Analyze changes**
2. **Assess quality**
3. **Decision:**
   - Good: SEND TELEGRAM MESSAGE
   - Needs improvement: Provide feedback

### Step 6: Provide Feedback (If needed)

```
operationId: "issues/create-comment"
parameters:
  body: "@copilot \n\nImplementation reviewed and found issues:\n\n1. <Specific issue 1>\n2. <Specific issue 2>\n\nPlease address these points."
```

**AFTER FEEDBACK:** SET NEXT 10-MINUTE TIMER!

### Step 7: FINAL TELEGRAM MESSAGE (ONLY WHEN COMPLETE!)

**Only when Copilot is finished and everything is good:**

**Method 1: Specific Session:**
```
sessions_send(
  sessionKey: "agent:main:telegram:direct:1196751676",
  message: "🚀 GitHub Issue finished!\n\nProject: <owner>/<repo>\nIssue: #<issue-id> - <title>\nStatus: Copilot finished\n\nReady for your final review!"
)
```

**Fallback if Method 1 fails:**
```
sessions_spawn(
  mode: "run",
  task: "Send Telegram message: <message>"
)
```

**Last resort:**
- Write notification in repo file
- Let user know it's finished
- NEVER GIVE UP!

## MONITORING LOGIC EXAMPLE

```
Every 10-minute check:

if copilot_not_finished:
    create_new_timer(10_minutes)
    # SILENT check - NO Telegram message!
else:
    send_telegram_message()  # ONLY when finished!
```

## Troubleshooting

| Problem | Solution |
|---|---|
| Monitor timer stops | Create new `at` timer every 10 min! |
| Telegram delivery fail | Try ALL methods above! Never give up! |
| Copilot not active | @copilot mention and assignment check - continue monitoring! |
| Assignment fails | Try alternative methods but never give up! |

## Safety Guidelines

- NEVER stop monitoring until Copilot is finished
- ONLY send Telegram message when FINISHED
- Always include [Auto-Generated] in issue title
- Always assign Copilot after issue creation
- Only use `at` scheduling for timers (but create multiple `at` jobs!)
- Clean up timers after completion
- Never modify existing issues without user permission

## FINAL REMINDER

**THIS SKILL REQUIRES:**
1. CONTINUOUS MONITORING - every 10 minutes until finished
2. TELEGRAM MESSAGE - only when finished

**NEVER GIVE UP! FIND A SOLUTION!**