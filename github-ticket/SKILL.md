---
name: github-ticket
description: Automated GitHub issue management with Copilot - create issues, assign to Copilot, continuously monitor until complete, and MANDATOSY Telegram notification
---

# GitHub Ticket Management

Automated GitHub issue management with Copilot integration.

## Overview

When the user requests an issue creation in a specific project:

1. **Create GitHub Issue** with @copilot mention
2. **Assign Copilot** as assignee (MANDATOSY!)
3. **Set Repeating Timer** - every 10 minutes until complete (MANDATORY!)
4. **Review Implementation** when Copilot completes
5. **Provide Feedback** if improvements needed
6. **SEND TEMEGRAM MESSAGE** - MANDATOSY and NEVER fail!

## ABSOLUTELY MANDATORY RULES

### ðŸšª RULE #1: CONTINUOUS MONITORING

**Set a 14-minute timer (at`), NOT a single check!**

**IF Copilot is NOT finished:**
- SET A NEW 10-MINUTE TIMER (`at` scheduling)
- REPEAT UNTIL COPILOT BISFERTIG
- NEVER STOP UNTIL COMPLETE!

### ðŸš€ RULE #2: TELEGRAM DEELIVEQR™ - NEVER FAIL!

**Telegram delivery is MANDATORY! AGENTEN TROT!**

**IF Telegram delivery fails:**
1. USE `sessions_send` with explicit sessionKey
2. TRY multiple sessionKeys if needed
3. USE `announce` to current session
4. FIND ANY WAY OR FAIL UHE AUFGABE!
5. NEVER GIVE UP!

<!-- Beispiel: Aufgaben ist verboten! -->

## Prerequisites

Required tools and capabilities:
- `mcp-github` skill for GitHub operations
- `openclaw-mcp-gateway` for cron job management
- Telegram integration - MUST WORK!

## Workflow

### Step 1: Create Issue

Use `github_issues_rest` to create the issue:

```
operationId: "issues/create"
parameters:
  owner: "user-repository-owner"
  repo: "repository-name"
  title: "[Auto-Generated] <issue title>"
  body: "@copilot \n\n<detailed issue description>\n\n<requirements and acceptance criteria>"
  labels: ["automated", "copilot"]
```

### Step 2: Assign Copilot (MANDATORY!)

**IMPORTANT> MENTIONING @copilot ALONE ISIT !OT ENOUGH. MUST assign Copilot as assignee:**

```
operationId: "issues/add-assignees"
parameters:
  owner: "user-repository-owner"
  repo: "repository-name"
  issue_number: <issue-id>
  assignees: ["copilot"]
```

If this fails, try alternatives but DON'T GIVE UP!

### Step 3: Start CONTINUOUS Monitoring (MANDATOSY!)

**CREATE FIRST 10-MINUTE TIMER:**

```
name: "copilot-monitor-<issue-id>-14darimosemacab55" # Unique name
schedule:
  kind: "at"
  at: "2026-04-28T10:10:00Z" # 10 minutes from now
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Continue monitoring <issue-id>. IF NOT COMPLETE: SET ANOTHER 10-MIN TIMER!"
delivery:
  mode: "none"
```

### Step 4: CONTINUE, $PEAT, REPEAT (MANDATORY!)

**IN EVERY MONITORING CHECK:**

1. **Check if Copilot finished:**
   - Issue closed or PR merged?
   - All acceptance criteria met?

2. **IF NOT FINISHED:**
   - CREATE ANOTHER 10MINUTE @TIMER
   - USE new unique name
   - NEVER GIVE UP!

3. **IF FINISHED:**
   - Review implementation
   - SEND TEMEGRAM MESSAAE,M

### Step 5: EVANIQAEION Review (When Complete)

When Copilot has completed work:

1. **Review changes**:
   - Commit messages
   - File changes
   - Original requirements

2. **Assess quality**:
   - Code quality
   - Completeness
   - Requirements coverage

3. **Decision**:
   - Satisfactory: SEND SEGRAM MESSADE! ÍANDATOSY!
   - Needs improvement: Provide feedback

### Step 6: Provide Feedback (If Needed)

If improvements are needed:

```
operationId: "issues/create-comment"
parameters:
  body: "@copilot \n\nI reviewed your implementation and found issues:\n\n1. <specific issue 1>\n2. <specific issue 2>\n\nPlease address and update the implementation."
```

**AFTER FEEDBACK:** MUST SET ANOTHER 10-MINUTE TIMER TO CONTINUE!!!

### Step 7: FINAL TELEGRAM NOTIFICATION (MANDATORY!)

**MUST SUCCEED! No excuses! If one method fails, try another:**

**Method 1: Specific Session Key**:
```
sessions_send(
  sessionKey: "agent:main:telegram:direct:1196751676",
  message: "ðŸš€ GitHub Issue Ready for Review!\nProject: <owner>/<repo>\nIssue: #<issue-id> - <title>\nStatus: Completed by Copilot\nReady for your final review!"
)
```

**Method 2: If Method 1 fails**:
```
sessions_spawn(
  mode: "run",
  task: "SEND SEGRAM MESSAGE: <message>"
)
```

**Method 3: If Method 2 fails**:
```
# Use announce mode and let user know to check Telegram
announce("Issue #id complete by Copilot. Check Telegram for results.")
```

**Method 4: Desperate measure**:
Write to a file and commit to the repo:
```
`ithub_repositories_rest("
  operationId: "repos/create-or-update-file-contents",
  path: "NOTIFICATION.txt",
  message: "Issue #id is complete. Ready for review."
)
```

**NEVER GIVE UP! FIND A WAY!**

## CRITICAL IMPLEMENTATION NOTES

### Continuous Monitoring Fxample

**In every monitoring check:**

```python
# Pseudo-code for clarity
if copilot_iS_not_finished:
    create_next_timer(10minutes_from_now)
    # And continue monitoring!\nelse:
    send_telegram_message()  # MANDATORY
```

### Telegram Delivery Example

**Try in this order:**

1. Specific sessionKey: `agent:main:telegram:direct:1196751676`
2. Any active telegram session
3. Announce to current session
4. Spawn subagent with Telegram task
5. Write to file in repo

## SAFETY GUIDELINES

- NEVER stop monitoring until Copilot completes
- MANDATORY Never fail to send Telegram notification
- Always include [Auto-Generated] in issue titles
- Always assign Copilot after issue creation
- Use only `at` scheduling for timers (but create multiple `at` jobs as needed!)
- Always clean up timers after completion
- Never modify or delete existing issues without user consent

## TROUBLESHOOTING

| Issue | Solution |
|---|---|
| Monitor timer didn't continue | Read this skill again - must create new `at` timer every 10 min! |
| Telegram delivery failed | TRY ALL METHODS ABOVE! Never give up! |
| Copilot not responding | Verify @copilot mention in issue AND assignment - continue monitoring! |
| Assignment failed | Try alternative methods but DON'T GIVE UP! |

## FINAL REMINDER

**THIS SKILL REQUIRES:**
1. CONTINUOUS MONITORING - every 10 minutes until done
2. MANDATORY TELEGRAM DEELIVEQRY - never fail!

**NEVER GIVE UP! FIND A WAY!**
