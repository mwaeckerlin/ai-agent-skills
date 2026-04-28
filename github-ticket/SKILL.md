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

**Telegram message ONLY when COMPLETION!**

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
- Telegram integration - MUST WORK!

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
  at: "2026-04-28T10:10:00Z" # 10 minutes from now
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Copilot monitoring for Issue <issue-id>. SILENT CHECK!"
delivery:
  mode: "none"  # NO Telegram message!
```

### Step 4: MONITORING LOGIC (REPEAT!)

**IN EVERY MONITORING CHECK:**

1. **Check Issue Status with issues/get API:**
   ```
   operationId: "issues/get"
   parameters: { owner, repo, issue_number }
   ```

2. **Determine Copilot Status:**
   
   **COPILOT HAS CREATED PR when:**
   - `data.linkedPullRequests.nodes.length > 0`
   - This means Copilot finished initial work
   
   **COPILOT NOT FINISHED when:**
   - `data.linkedPullRequests.nodes.length === 0`
   - Keep monitoring

3. **If NOT FINISHED (no PR yet):**
   - CREATE NEW 10-MINUTE TIMER
   - Use unique names (-2, -3, etc.)
   - NEVER STOP!

4. **If PR EXISTS:**
   - Proceed to Step 5: Review PR

### Step 5: PR Review Process

**When linkedPullRequests.nodes.length > 0:**

1. **Get PR Details:**
   ```
   Get first PR from linkedPullRequests.nodes[0]
   PR number: data.linkedPullRequests.nodes[0].number
   ```

2. **Review PR Implementation:**
   - Check code quality
   - Verify requirements met
   - Assess completeness

3. **Decision:**
   
   **If Implementation is GOOD:**
   - Proceed to Step 7: Send Telegram
   
   **If Implementation needs IMPROVEMENT:**
   - Proceed to Step 6: Provide Feedback

### Step 6: Provide PR Feedback

**Comment on the PR (NOT the issue):**

```
operationId: "issues/create-comment"
parameters:
  owner: <owner>
  repo: <repo>
  issue_number: <PR number>
  body: "@copilot\n\nImplementation review feedback:\n\n1. <Specific issue 1>\n2. <Specific issue 2>\n\nPlease address these points."
```

**AFTER FEEDBACK:**
- SET NEW 10-MINUTE TIMER
- Return to Step 4 (monitoring)
- Continue until implementation is satisfactory

### Step 7: FINAL TELEGRAM NOTIFICATION

**Send ONLY when PR is reviewed and implementation is satisfactory:**

**Method 1: Direct Session Message:**
```
sessions_send(
  sessionKey: "agent:main:telegram:direct:1196751676",
  message: "✅ GitHub Issue READY for your review!\n\nProject: <owner>/<repo>\nIssue: #<issue-id> - <title>\nPR: #<pr-number>\n\nCopilot implementation reviewed and approved.\nReady for your final review and merge!"
)
```

**Method 2: Spawn if Method 1 fails:**
```
sessions_spawn(
  mode: "run",
  task: "Send Telegram: Issue <issue-id> ready for review"
)
```

**Method 3: Write notification file:**
```
Write to: NOTIFICATION.md
Content: Issue #<id> ready for review at <timestamp>
```

**ENSURE TELEGRAM IS SENT - TRY ALL METHODS!**

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