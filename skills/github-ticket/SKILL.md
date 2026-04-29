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

When finished (`state === "closed"` OR `merged === true`):
- ✅ Send Telegram message (already sent when review was left open in step 6 above)

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

## Copilot Task Status — State Machine

Check every 10 minutes. This is a state machine with two states:

### State: COMPLETED (terminal — stop all timers)

**Condition:** `state === "closed"` OR `merged === true`

This is the **one and only finish criterion**. Once reached, cancel all timers and stop monitoring.

### State: REVIEW REQUESTED (action required)

**Condition:** `requested_reviewers` is non-empty OR `requested_teams` is non-empty

When this condition is met, perform the following review steps **in order**:

1. **Read the PR body** (`body` field) carefully.
2. **Review the ticket** — check the implementation against the issue requirements.
3. **Run all tests** — execute the project's test suite.
4. **If issues are found or tests fail:**
   - Submit a review with **Reject** (request changes).
   - Add a comment describing exactly what needs to be fixed.
   - Set a new 10-minute timer and re-check later.
5. **If tests pass but `!mergeable || mergeable_state !== "clean"`:**
   - Submit a review with **Reject** (request changes).
   - Add a comment asking to make the branch mergeable (resolve conflicts, fix CI, rebase, etc.).
   - Set a new 10-minute timer and re-check later.
6. **If tests pass AND `mergeable === true` AND `mergeable_state === "clean"`:**
   - Do **not** reject, do **not** approve.
   - Leave the review **open** with a comment: _"looks like ready to merge"_ plus your findings (do not say "ready to merge" — that is the user's decision).
   - Post a Telegram message that the PR is ready for the user to review, **including the PR URL**.
   - Set a new 10-minute timer and re-check later.

> **After steps 4, 5, or 6:** always set a new timer and continue monitoring until COMPLETED.

### Still waiting (no state matched)

If no PR exists yet, or the PR exists but `requested_reviewers` and `requested_teams` are both empty (Copilot has not requested a review yet):
- Set a new 10-minute timer and re-check later. No action, no Telegram message.

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

**In every monitoring check, evaluate the state machine in order:**

#### COMPLETED (terminal — stop all timers)

```
operationId: "pulls/get"
parameters: { owner, repo, pull_number }
```

If `state === "closed"` OR `merged === true` → **COMPLETED**. Stop all timers. No further action needed.

#### REVIEW REQUESTED

If `requested_reviewers` is non-empty OR `requested_teams` is non-empty:

1. **Read the PR body** carefully (`body` field).

2. **Review the ticket** — compare implementation against the original issue requirements.

3. **Run all tests** for the repository.

4. **If issues are found or tests fail:**
   ```
   operationId: "pulls/create-review"
   parameters:
     owner: <owner>
     repo: <repo>
     pull_number: <PR number>
     event: "REQUEST_CHANGES"
     body: "<Exact description of what needs to be fixed>"
   ```
   Set a new 10-minute timer and continue monitoring.

5. **If tests pass but `!mergeable || mergeable_state !== "clean"`:**
   ```
   operationId: "pulls/create-review"
   parameters:
     owner: <owner>
     repo: <repo>
     pull_number: <PR number>
     event: "REQUEST_CHANGES"
     body: |
       The branch is not mergeable yet.

       Current state: mergeable=<value>, mergeable_state=<value>

       Please resolve:
       - `dirty`: fix merge conflicts with the base branch.
       - `blocked`: ensure required checks pass and reviews are approved.
       - `behind`: rebase or merge the base branch into this branch.
       - `unstable`: fix failing CI checks.
   ```
   Set a new 10-minute timer and continue monitoring.

6. **If tests pass AND `mergeable === true` AND `mergeable_state === "clean"`:**
   - Do **not** reject. Do **not** approve. Leave the review **open**.
   ```
   operationId: "pulls/create-review"
   parameters:
     owner: <owner>
     repo: <repo>
     pull_number: <PR number>
     event: "COMMENT"
     body: "looks like ready to merge\n\n<Your findings>"
   ```
   - Then send a Telegram message that the PR is ready for the user to review, **including the PR URL**:
   ```
   sessions_send(
     sessionKey: "agent:main:telegram:direct:1196751676",
     message: "👀 PR ready for your review!\n\nProject: <owner>/<repo>\nPR: #<pr-number> — <title>\nURL: <pr_html_url>\n\nImplementation reviewed. Looks like ready to merge."
   )
   ```
   Set a new 10-minute timer and continue monitoring.

#### WAITING (no action)

No PR exists yet, or PR exists but both `requested_reviewers` and `requested_teams` are empty:
- Create a new 10-minute timer.
- No Telegram message, no review action.

## Monitoring Logic Example

```
Every 10-minute check:

# COMPLETED — terminal state, stop all timers
if state == "closed" or merged == true:
    stop_all_timers()
    return

# REVIEW REQUESTED — Copilot handed off to reviewer
elif requested_reviewers or requested_teams:
    read_pr_body()
    review_ticket_requirements()
    run_all_tests()

    if issues_found or tests_failed:
        submit_review(verdict="reject", comment="<what needs to be fixed>")
        create_new_timer(10_minutes)

    elif not mergeable or mergeable_state != "clean":
        submit_review(verdict="reject", comment="<make the branch mergeable>")
        create_new_timer(10_minutes)

    else:
        # Do NOT approve, do NOT reject — leave review open
        post_review_comment("looks like ready to merge\n\n<findings>")
        send_telegram("PR is ready for your review: <PR URL>")
        create_new_timer(10_minutes)

# WAITING — no PR yet, or reviewer not yet assigned
else:
    create_new_timer(10_minutes)
    # Silent check — no Telegram message
```

## Troubleshooting

| Problem | Solution |
|---|---|
| Monitor timer stops | Create a new `at` timer every 10 minutes. |
| Telegram delivery fails | Try all methods listed above in order. |
| Copilot not active | Verify @copilot mention and assignment, then continue monitoring. |
| Assignment fails | Try alternative assignment methods and continue. |
| `mergeable_state` is `"unknown"` | GitHub is still computing — wait for next 10-minute check. |
| PR has reviewer assigned but not clean | Submit reject review (Step 4, item 5) and keep monitoring. |

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
