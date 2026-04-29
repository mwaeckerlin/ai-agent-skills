---
name: github-ticket
description: Automated GitHub project and issue management with Copilot - create projects, create issues, assign to Copilot, continuously monitor, review implementation, and coordinate with user for final merge
---

# GitHub Ticket Skill

The skill applies when the user:
- asks to **check**, **monitor**, or **watch** a PR
- asks to **check**, **monitor**, or **watch** an Issue
- asks to **create an Issue** in a project
- asks to **create a new project**

## Skill Flow

The skill follows this state machine:

```plantuml
[*] --> pr : User asks to\nmonitor a PR
[*] --> issue : User asks to\ncreate an Issue
[*] --> check_issue : User asks to\nmonitor an issue
[*] --> project : User asks to\ncreate a new project

state "Project" as project
project: create a new project with a README
project: make sure the project is initialized

state "Issue" as issue
issue: create issue according to request
issue: assign issue to copilot

state "Timer" as timer
timer: setup a timer to expire in 10 minutes

state "Load Issue Details" as check_issue
check_issue: check if PR has been created

project --> issue : create an issue\nto implement the project\naccording to User request

issue --> timer
timer --> pr : timeout and\nPR is known
timer --> check_issue : timeout and\nPR is unknown
check_issue --> timer : no PR created yet
check_issue --> pr : PR created

state "Review Process" as rp {
  state "Load PR Details" as pr
  pr: Use mcp-github Skill to read PR

  state "Clone or Pull\nRepository" as pull
  pull: clone repository (if not locally available)
  pull: checkout PR branch
  pull: pull latest changes from origin

  state "Review" as review
  review: read PR body information
  review: carefully review code changes
  review: check for best practices and clean code
  review: check for security-relevant issues
  review: review test cases, ask for more tests if necessary

  state "Test" as test
  test: run all test cases
  test: standard is npm run test if package.json present
  test: reject if npm run test does not run all tests

  state "Reject" as reject
  reject: reject review with comment
  reject: describe the problem to be fixed
  reject: mention @copilot

  state "Accept" as accept
  accept: write review comment: "looks good to me"
  accept: do not formally "accept", do not "reject"

  state "Notification" as notify
  notify: send message to User that process is finished
  notify: add link to PR in message
  notify: wait for user response

  state "Accepted" as accepted
  accepted: accept the PR review
  accepted: merge PR
  accepted: delete branch

  pr --> pull : requested_reviewers not empty\nor\nrequested_teams not empty
  pr --> timer : requested_reviewers is empty\nand\nrequested_teams is empty
  pull --> review
  review --> test : review passed
  review --> reject : issues found
  test --> accept : tests pass\nand mergeable_state == "clean"
  test --> reject : tests fail
  test --> reject : tests pass but\nmergeable_state != "clean"\nask to cleanup
  reject --> timer
  accept --> notify
  notify --> reject : negative user response\nwith reason to reject
  notify --> accepted : positive user response\nsuch as "accept" or "merge"
}

pr --> [*] : state == "closed"\nor merged == true
accepted --> [*]
```

## State Descriptions

### Project

When the user asks to create a new project:

1. Create the repository with a `README.md` so it is properly initialized (empty repositories cannot be used by Copilot).
2. Proceed to the **Issue** state to create an implementation issue.

```
operationId: "repos/create-for-authenticated-user"
parameters:
  name: "<repo-name>"
  description: "<project description>"
  auto_init: true
```

> **Note:** If no account/owner is specified, always use the user's own account. Never guess or infer an owner from the repository name.

### Issue

Create a GitHub issue and assign it to Copilot:

1. **Create the issue** — include a detailed description and acceptance criteria. Always mention `@copilot` in the body:

```
operationId: "issues/create"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  title: "[Auto-Generated] <Issue Title>"
  body: "@copilot\n\n<Detailed description>\n\n<Requirements and acceptance criteria>"
  labels: ["automated", "copilot"]
```

2. **Assign Copilot** as assignee — the `@copilot` mention alone is not enough:

```
operationId: "issues/add-assignees"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  issue_number: <issue-id>
  assignees: ["copilot"]
```

3. Proceed to the **Timer** state.

### Timer

Set up a timer to expire in 10 minutes, then re-enter the state machine:

```
name: "copilot-monitor-<issue-id>-<sequence>" # unique name per check
schedule:
  kind: "at"
  at: "<now + 10 minutes>"
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Check Copilot status for issue #<issue-id> in <owner>/<repo>. Continue the github-ticket skill state machine."
delivery:
  mode: "none"
```

### Load Issue Details (check_issue)

When monitoring an issue (no PR known yet):

1. Load the issue details using the `mcp-github` skill.
2. Search for any PR that references this issue (e.g. via `closes #<issue-id>` in the PR body or via linked PRs).
3. **If a PR has been created** → proceed to **Load PR Details**.
4. **If no PR exists yet** → proceed to **Timer** and re-check later.

### Load PR Details (pr)

Read the current PR state using the `mcp-github` skill:

```
operationId: "pulls/get"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  pull_number: <pr-number>
```

**Terminal check — evaluate first:**
If `state === "closed"` OR `merged === true` → **process is finished**. Stop all timers. Do nothing else. This is the exit point.

**Reviewer check:**
- If `requested_reviewers` is non-empty OR `requested_teams` is non-empty → proceed to **Clone or Pull Repository**.
- Otherwise → proceed to **Timer** (Copilot has not yet requested a review).

### Clone or Pull Repository (pull)

Before reviewing, get the latest code locally:

1. If the repository is not yet cloned locally, clone it.
2. Check out the PR branch.
3. Pull the latest changes from origin.

```bash
git clone https://github.com/<owner>/<repo>.git   # if not yet cloned
cd <repo>
git fetch origin
git checkout <pr-branch>
git pull origin <pr-branch>
```

Proceed to **Review**.

### Review

Perform a thorough code review:

1. **Read the PR body** (`body` field) — Copilot leaves information about its tasks, implementation decisions, and reasons here.
2. **Review the code changes** carefully — check every changed file.
3. **Check for best practices and clean code** — naming, structure, readability, maintainability.
4. **Check for security-relevant issues** — input validation, authentication, injection risks, secrets in code, etc.
5. **Review the test cases** — check coverage and quality; ask for more tests if necessary.

**If the review finds issues** → proceed to **Reject**, describing every problem found.

**If the review passes** → proceed to **Test**.

### Test

Run the project's test suite against the checked-out PR branch:

- Standard: run `npm run test` if a `package.json` is present.
- If `npm run test` exists but does not run all tests, treat this as a failure and request fixes.
- Adapt to the project's tooling as appropriate (e.g. `make test`, `pytest`, `go test ./...`).

**If tests fail** → proceed to **Reject**, describing the failing tests.

**If tests pass but `mergeable_state !== "clean"`** → proceed to **Reject**, asking to fix mergeability (resolve conflicts, fix CI, rebase, etc.).

**If tests pass and `mergeable_state === "clean"`** → proceed to **Accept**.

> **Note:** `mergeable_state` may be `"unknown"` immediately after a push while GitHub computes it. If `"unknown"`, proceed to **Timer** and re-check later.

### Reject

Submit a `REQUEST_CHANGES` review, describing exactly what Copilot must fix:

```
operationId: "pulls/create-review"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  pull_number: <pr-number>
  event: "REQUEST_CHANGES"
  body: |
    <Exact description of every issue found.>

    @copilot please fix the above issues.
```

Proceed to **Timer** to re-check after 10 minutes.

### Accept

The review and tests have passed. Leave a comment — **do not formally approve, do not reject**:

```
operationId: "pulls/create-review"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  pull_number: <pr-number>
  event: "COMMENT"
  body: "looks good to me\n\n<Summary of findings / what was checked>"
```

Proceed to **Notification**.

### Notification

Inform the user that the PR is ready and wait for their decision:

Send a Telegram message including the PR URL:

```
sessions_send(
  sessionKey: "agent:main:telegram:direct:<user-id>",
  message: "✅ PR ready for your review!\n\nProject: <owner>/<repo>\nPR: #<pr-number> — <title>\nURL: <pr_html_url>\n\nCode reviewed and tests pass. Looks good to me.\n\nReply 'accept' or 'merge' to merge, or describe what should be changed."
)
```

**Wait for user response:**
- **Positive response** (e.g. "accept", "merge", "yes", "looks good") → proceed to **Accepted**.
- **Negative response with reason** → proceed to **Reject**, using the user's feedback as the rejection comment.

### Accepted

The user has approved. Finalize the PR:

1. **Accept the PR review** (formally approve):

```
operationId: "pulls/create-review"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  pull_number: <pr-number>
  event: "APPROVE"
  body: "Approved as requested by user."
```

2. **Merge the PR:**

```
operationId: "pulls/merge"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  pull_number: <pr-number>
  merge_method: "squash"
```

3. **Delete the branch:**

```
operationId: "git/delete-ref"
parameters:
  owner: "<owner>"
  repo: "<repo>"
  ref: "heads/<pr-branch>"
```

Process is complete. **→ [*]** (terminal state)

## Available GitHub PR State Fields

When evaluating a PR, the following fields are available:

| Field | Type | Values / Notes |
|---|---|---|
| `draft` | boolean | `true` = PR created as draft. Not a reliable signal of Copilot completion — treat as informational only. |
| `state` | string | `"open"`, `"closed"` |
| `merged` | boolean | `true` = already merged |
| `mergeable` | boolean/null | `true` = no conflicts, `false` = has conflicts, `null` = not yet computed |
| `mergeable_state` | string | `"clean"` (ready), `"dirty"` (conflicts), `"blocked"` (checks/approvals missing), `"behind"` (branch behind base), `"unstable"` (checks failed/pending), `"unknown"` (not yet determined) |
| `requested_reviewers` | array | reviewers not yet responded |
| `requested_teams` | array | teams not yet responded |
| `body` | string/null | PR description (Copilot summarises its work here) |
| `head.sha` | string | latest commit SHA on the PR branch |

## Error Handling

If any step cannot be completed (e.g. API error, test runner unavailable, unable to post a review):

- Increment the **error counter** for this monitoring cycle.
- Set a new 10-minute timer to retry later.
- After **3 consecutive errors**, post a Telegram message explaining the problem and the step that failed, so the user can investigate.
- Reset the error counter after a successful check.

## Safety Guidelines

- Always include `[Auto-Generated]` in the issue title.
- Always assign Copilot after issue creation — a `@copilot` mention alone is not enough.
- Use only `at` scheduling for timers.
- Never modify existing issues without user permission.
- Never stop the monitoring loop until the process reaches a terminal state (`[*]`).
