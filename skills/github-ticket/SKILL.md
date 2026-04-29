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

**Note:** Copilot may leave the PR in `draft: true` state even after finishing its work. **Do not use `draft` as a completion criterion.** A draft PR is still a valid completed PR for monitoring purposes.

## Available GitHub PR State Fields

When evaluating a PR, the following fields are available and should be considered:

| Field | Type | Values / Notes |
|---|---|---|
| `draft` | boolean | `true` = PR created as draft; `false` = explicitly marked ready for review. **Not a reliable signal of Copilot completion** — Copilot may finish work while `draft` is still `true`. Treat as informational only. |
| `state` | string | `"open"`, `"closed"` |
| `merged` | boolean | `true` = already merged |
| `mergeable` | boolean/null | `true` = no conflicts, `false` = has conflicts, `null` = not yet computed |
| `mergeable_state` | string | One of: `"clean"` (ready to merge), `"dirty"` (merge conflicts), `"blocked"` (required checks/approvals missing), `"behind"` (branch is behind base), `"unstable"` (some checks failed/pending), `"has_hooks"` (pre-receive hooks pending), `"unknown"` (not yet determined) |
| `review_decision` | string (GraphQL) | `"APPROVED"`, `"REVIEW_REQUIRED"`, `"CHANGES_REQUESTED"` |
| `auto_merge` | object/null | non-null if auto-merge is enabled |
| `requested_reviewers` | array | reviewers not yet responded |
| `requested_teams` | array | teams not yet responded |
| `labels` | array | labels attached to the PR (may be empty) |
| `body` | string/null | PR description body (Copilot typically summarises its work here) |
| `commits` | number | number of commits on the PR branch |
| `changed_files` | number | number of files changed |
| `updated_at` | string (ISO 8601) | timestamp of the last update (push, comment, etc.) — useful for detecting recent Copilot activity |
| `head.sha` | string | latest commit SHA on the PR branch |
| `statuses_url` | string | URL to query combined commit status (CI) |

> **Note:** `mergeable_state` may be `"unknown"` immediately after a push while GitHub computes it. Retry after a short delay if it is `"unknown"`.

## Copilot Task Status Definitions

**CRITICAL:** Understand when Copilot tasks are actually finished:

- **READY FOR REVIEW** = PR exists AND `state: "open"` (regardless of `draft` value)
- **FINISHED** = READY FOR REVIEW AND `mergeable_state: "clean"`
- **NEEDS CLEANUP** = READY FOR REVIEW AND `mergeable_state` is NOT `"clean"` (e.g. `"dirty"`, `"blocked"`, `"behind"`, `"unstable"`)
- **NOT FINISHED** = No PR exists yet

**Status checking workflow:**
1. Check if a PR linked to the issue exists
2. If no PR → NOT FINISHED, keep monitoring
3. If PR exists and `state: "open"` (ignore `draft` value):
   a. If `mergeable_state: "clean"` → **FINISHED**
   b. If `mergeable_state` is `"unknown"` → wait and re-check (GitHub is still computing); after **5 consecutive unknown checks** (~10 minutes), treat as NEEDS CLEANUP and post @copilot comment
   c. Otherwise → **NEEDS CLEANUP** — post @copilot comment (see Step 4b below) and keep monitoring

**Common mistakes to avoid:**
- ❌ Using `mergeable_state: "clean"` as the *only* criterion — it may be `"unknown"` right after a push
- ❌ Thinking `"merged": false` means "not finished" — an unmerged PR can still be ready
- ❌ Using `"draft": true` as "not finished" — Copilot finishes work while the PR is still a draft
- ✅ READY FOR REVIEW = PR exists + `state: "open"` (draft is irrelevant)
- ✅ FINISHED = READY FOR REVIEW + `mergeable_state: "clean"`

**Repository creation guidance:**
- **Problem:** Empty repositories (no base branch) cannot be used by Copilot
- **Solution:** Always initialize repositories with content OR pass Copilot task during creation
- Never create empty repository first, then assign Copilot separately

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

   **Step 2a — PR exists?**
   - If `data.linkedPullRequests.nodes.length === 0` → NOT FINISHED, create new timer and stop here.

   **Step 2b — PR is open. Check mergeable state** (ignore `draft` value):
   - `mergeable_state: "clean"` → **FINISHED**, proceed to check Copilot comment
   - `mergeable_state: "unknown"` → GitHub is still computing; create a 2-minute timer and recheck (up to 5 retries; if still unknown after 5 retries, treat as NEEDS CLEANUP)
   - Any other value (`"dirty"`, `"blocked"`, `"behind"`, `"unstable"`) → **NEEDS CLEANUP** — run Step 4b

3. **If a PR exists and FINISHED, check for Copilot's comment:**
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

### Step 4b: Handle NEEDS CLEANUP (Ready for Review but Not Clean)

**When the PR is `draft: false` but `mergeable_state` is NOT `"clean"`:**

Post a comment on the PR mentioning @copilot and explaining what needs to be fixed:

```
operationId: "issues/create-comment"
parameters:
  owner: <owner>
  repo: <repo>
  issue_number: <PR number>
  body: |
    @copilot

    This PR is marked as ready for review, but it cannot be merged yet.

    Current `mergeable_state`: **<mergeable_state value>**

    Please resolve the issue:
    - If `dirty`: fix merge conflicts with the base branch.
    - If `blocked`: ensure all required status checks pass and any required reviews are approved.
    - If `behind`: rebase or merge the base branch into this branch.
    - If `unstable`: check failing CI checks and fix them.
    - If `unknown` (persistent): check if there is an underlying issue preventing GitHub from computing the state.

    Once resolved, the PR should reach `mergeable_state: "clean"`.
```

**After posting the comment:**
- Create a new 10-minute timer (continue monitoring)
- Do NOT send a Telegram notification

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

if no_pr_exists or pr_is_draft:
    create_new_timer(10_minutes)
    # Silent check — no Telegram message

elif mergeable_state == "unknown" and unknown_retry_count < 5:
    unknown_retry_count += 1
    create_new_timer(2_minutes)   # Recheck soon while GitHub computes

elif mergeable_state != "clean":
    # Includes persistent "unknown" after 5 retries
    post_copilot_comment(explain_what_to_fix)   # @copilot mention
    create_new_timer(10_minutes)

elif not copilot_has_explanatory_comment:
    create_new_timer(10_minutes)

else:  # ready, clean, and Copilot has commented
    review_pr()
    post_approval_comment()
    send_telegram_message()
```

## Troubleshooting

| Problem | Solution |
|---|---|
| Monitor timer stops | Create a new `at` timer every 10 minutes. |
| Telegram delivery fails | Try all methods listed above in order. |
| Copilot not active | Verify @copilot mention and assignment, then continue monitoring. |
| Assignment fails | Try alternative assignment methods and continue. |
| `mergeable_state` is `"unknown"` | GitHub is still computing — set a 2-minute timer and recheck. |
| PR is ready for review but not clean | Post a @copilot comment (Step 4b) and keep monitoring. |

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
