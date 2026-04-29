---
name: github-ticket
description: Automated GitHub issue management with Copilot - create issues, assign to Copilot, continuously monitor until complete, and send Telegram notification only when finished
---

# GitHub Ticket Management

Automate GitHub issue management with Copilot integration.

## Overview

When the user requests issue creation in a specific project:

1. **Initialize project** if it is new (see "New Project Initialization" below)
2. **Create GitHub issue** with @copilot mention
3. **Assign Copilot** as assignee (mandatory)
4. **Start monitoring** — every 10 minutes until complete
5. **Review implementation** when Copilot completes
6. **Provide feedback** if improvements are needed
7. **Send Telegram message** — only when complete

## Mandatory Rules

### Rule 0: Default Repository Owner

**If no account/owner is specified, always use the user's own account.**

- Never guess or infer an owner from the repository name.
- Example: if the user's account is `YOUR_USERNAME` and the task says "create an issue in `openclaw`", the repository is `YOUR_USERNAME/openclaw` — **not** `openclaw/openclaw` or any other owner.

### Rule 1: Continuous Monitoring

**Set a new 10-minute timer after every check — never stop.**

As long as Copilot is not finished:
- Create a new 10-minute timer (`at` scheduling)
- Repeat until Copilot is finished
- Never stop before completion

### Rule 2: Telegram Message — Only When Finished

**Send a Telegram message only upon completion.**

During monitoring:
- Do not send Telegram messages during intermediate checks
- Perform a silent check and continue monitoring

When finished (all of the following must be true):
- ✅ A PR linked to the issue exists
- ✅ Copilot has left at least one new comment on the PR explaining what was done and how
- ✅ Implementation has been reviewed
- 📱 Send Telegram message

**Note:** Draft status of an issue or PR is **not** a completion criterion. A draft PR still counts as a valid PR for monitoring purposes.

## Prerequisites

Required tools and capabilities:
- `mcp-github` skill for GitHub operations
- `openclaw-mcp-gateway` for cron job management
- Telegram integration (must be configured and operational)

## Workflow

### Step 0: New Project Initialization

**If the task involves a new (empty) repository that has not been initialized yet:**

**Preferred method — assign directly to Copilot via the issue:**

Create the issue with an initialization task description and assign Copilot directly. This is more efficient than first generating files manually and then creating a ticket.

```
operationId: "issues/create"
parameters:
  owner: "<user-account>"
  repo: "<repo-name>"
  title: "[Auto-Generated] Initialize project"
  body: "@copilot\n\nPlease initialize this repository:\n\n<Description of the project and required setup>\n\nAt minimum, create a README.md and any other standard project scaffolding."
  labels: ["automated", "copilot"]
```

Then assign Copilot as described in Step 2 and continue with the normal workflow.

**Fallback method — generate README.md manually:**

Only use this if the preferred method is not applicable. Create a minimal `README.md` directly via the API to initialize the repository before creating further issues.

---

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

### Step 2: Assign Copilot (mandatory)

**Important:** @copilot mention alone is not enough — you must also assign Copilot as an assignee:

```
operationId: "issues/add-assignees"
parameters:
  owner: "repo-owner"
  repo: "repo-name"
  issue_number: <issue-id>
  assignees: ["copilot"]
```

### Step 3: Start Monitoring (silent)

**First 10-minute timer:**

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
  mode: "none"  # No Telegram message
```

### Step 4: Monitoring Logic

**In every monitoring check:**

1. **Check Issue Status with issues/get API:**
   ```
   operationId: "issues/get"
   parameters: { owner, repo, issue_number }
   ```

2. **Determine Copilot Status:**
   
   **Copilot has finished when ALL of the following are true:**
   - A PR linked to the issue exists (`data.linkedPullRequests.nodes.length > 0`)
   - Copilot has left at least one new comment on the PR explaining what was done and how
   
   **Draft status is NOT a criterion** — a draft PR still counts as a valid PR.
   
   **Copilot is not finished when:**
   - No linked PR exists (`data.linkedPullRequests.nodes.length === 0`), **or**
   - A PR exists but Copilot has not yet left an explanatory comment on it
   - Keep monitoring

3. **If a PR exists, check for Copilot's comment:**
   ```
   operationId: "issues/list-comments"
   parameters: { owner, repo, issue_number: <PR number> }
   ```
   - Look for a comment by `copilot` (or `github-copilot[bot]`) that explains what was done and how.
   - If no such comment exists yet, Copilot is not finished — create a new 10-minute timer.

4. **If not finished (no PR yet, or PR exists but no Copilot comment):**
   - Create a new 10-minute timer
   - Use unique names (-2, -3, etc.)
   - Never stop before completion

5. **If a PR exists and Copilot has left an explanatory comment:**
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
   
   **If implementation is good:**
   - Proceed to Step 5.5: Write Approval Comment
   - THEN proceed to Step 7: Send Telegram
   
   **If implementation needs improvement:**
   - Proceed to Step 6: Provide Feedback

### Step 5.5: Write Approval Comment (MANDATORY!)

**CRITICAL: This MUST be done BEFORE sending Telegram!**

**Comment on the PR without @copilot mention:**

```
operationId: "issues/create-comment"
parameters:
  owner: <owner>
  repo: <repo>
  issue_number: <PR number>
  body: "**Implementation Review Complete**\n\nReviewed components:\n- Code quality: <assessment>\n- Requirements: <assessment>\n- Tests: <assessment>\n\nReady for merge."
```

**VERIFY the comment was posted successfully.**
**If comment fails: DO NOT proceed to Step 7. Retry or notify user.**

### Step 6: Provide PR Feedback

**Comment on the PR (not the issue):**

```
operationId: "issues/create-comment"
parameters:
  owner: <owner>
  repo: <repo>
  issue_number: <PR number>
  body: "@copilot\n\nImplementation review feedback:\n\n1. <Specific issue 1>\n2. <Specific issue 2>\n\nPlease address these points."
```

**After providing feedback:**
- Set a new 10-minute timer
- Return to Step 4 (monitoring)
- Continue until implementation is satisfactory

### Step 7: Final Telegram Notification

**Send ONLY after Step 5.5 (Approval Comment) is complete:**

**CRITICAL: Do NOT send Telegram if approval comment was not posted!**

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

**Try all methods above in order to ensure the Telegram message is sent.**

## Monitoring Logic Example

```
Every 10-minute check:

if copilot_not_finished:
    create_new_timer(10_minutes)
    # Silent check — no Telegram message
else:
    send_telegram_message()  # Only when finished
```

## Troubleshooting

| Problem | Solution |
|---|---|
| Monitor timer stops | Create a new `at` timer every 10 minutes. |
| Telegram delivery fails | Try all methods listed above in order. |
| Copilot not active | Verify @copilot mention and assignment, then continue monitoring. |
| Assignment fails | Try alternative assignment methods and continue. |

## Safety Guidelines

- **Never** stop monitoring until Copilot is finished
- Only send the Telegram message when finished
- Always include [Auto-Generated] in issue title
- Always assign Copilot after issue creation
- Only use `at` scheduling for timers; create multiple sequential `at` jobs as needed
- Clean up timers after completion
- Never modify existing issues without user permission

## Final Reminder

**This skill requires:**
1. **Continuous monitoring** — every 10 minutes until finished
2. **Telegram message** — only when finished

**Always persist until a solution is found.**
