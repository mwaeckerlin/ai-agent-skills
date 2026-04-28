---
name: github-ticket
description: Automated GitHub issue management with Copilot - create issues, monitor progress, review implementation, and deliver feedback
---

# GitHub Ticket Management

Automated GitHub issue management with Copilot integration.

## Overview

When the user requests an issue creation in a specific project:

1. **Create GitHub Issue** with @copilot mention
2. **Monitor Progress** using timed checks
3. **Review Implementation** when Copilot completes
4. **Provide Feedback** if improvements needed
5. **Notify User** via Telegram when ready for review

## Prerequisites

Required tools and capabilities:
- `mcp-github` skill for GitHub operations
- `openclaw-mcp-gateway` for cron job management
- Telegram integration for notifications

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

### Step 2: Set Monitoring Timer

Use `openclaw_cron_add` to create a timed check: 

```
name: "copilot-issue-monitor-<issue-id>"
schedule:
  kind: "at"
  at: "2024-12-31T10:10:00Z" # 10 minutes from now
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Check GitHub issue -<issue-id> for Copilot progress"
delivery:
  mode: "none" # Silent monitoring
```

### Step 3: Monitoring Logic

In each monitoring check:

1. **Check Issue Status**:
   ```
   operationId: "issues/get"
   parameters: { owner, repo, issue_number }
   ```

2. **Check for Pull Requests**:
   ```
   operationId: "pulls/list"
   parameters: { owner, repo, state: "open" }
   ```

3. **Check for Recent Commits**:
   ```
   operationId: "repos/list-commits"
   parameters: { owner, repo, since: "<timestamp>" }
   ```

### Step 4: Progress Assessment

Determine Copilot status:

- **No Activity**: Schedule next check in 10 minutes
- **In Progress**: Pull request exists but not merged
- **Completed**: Issue closed or PR merged - proceed to review

### Step 5: Implementation Review

When Copilot has completed work:

1. **Analyze Changes**:
   - Review commit messages
   - Examine diffs and file changes
   - Check against original requirements

2. **Assess Quality**:
   - Code quality and best practices
   - Completeness of implementation
   - Adherence to specifications

3. **Decision**:
   - If satisfactory: Proceed to user notification
   - If needs improvement: Provide feedback
!## Step 6: Provide Feedback

If improvements are needed:

```
operationId: "pulls/create-review" or "issues/create-comment"
parameters:
  body: "@copilot \n\nI reviewed your implementation and found the following issues:\n\n1. <specific issue 1>\n2. <specific issue 2>\n\nPlease address these points and update the implementation."
```

### Step 7: User Notification

When everything is ready for review: 

```
sessions_send(to = "telegram"):
"🚀 GitHub Issue Ready for Review!
\nProject: <owner>/<repo>\nIssue: #<issue-id> - <title>\nStatus: Completed by Copilot\n\n✅ Implementation has been reviewed and approved\n✅ All requirements met\n\nReady for your final review!"
```

## Implementation Details

### Issue Creation Template

```markdown
## Issue Title
[Auto-Generated] <descriptive title>

## Description
@copilot

\n<detailed description of what needs to be done>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Additional Notes
<any additional context or specific requirements>
```

### Monitoring Parameters

- **Initial Check**: 10 minutes after issue creation
- **Recurring Checks**: Every 10 minutes until completion
- **Max Checks**: 24 hours (auto-cancel after)
- **Timeout**: 5 minutes per check

### Quality Assessment Criteria

1. **Code Quality**:
   - Clear, readable code
   - Appropriate comments
   - Follows project conventions

2. **Completeness**:
   - All requirements addressed
   - Acceptance criteria met
   - No major functionality gaps

3. **Testing**:
   - Tests included where appropriate
   - Existing tests still pass
   - No breaking changes

## Example Usage

```markdown
### User Request
User: "Please create an issue in mwaeckerlin/my-project to add a new README section explaining the installation process."

### Agent Execution
1. Creates GitHub issue with @copilot mention
2. Sets up 10-minute timer
3. Monitors progress automatically
4. Reviews Copilot's implementation
5. Notifies user when ready
```

## Troubleshooting

| Issue | Solution |
|---|---|
| Cron job failed | Check `openclaw_cron_runs` for error details |
| Copilot not responding | Verify @copilot mention in issue |
| Telegram notification failed | Check Telegram session and delivery config |
| Review failed | Ensure mcp-github skill is available and functional |

## Safety Guidelines

- Do not create issues without explicit user request
- Always include [Auto-Generated] in issue titles
- Use only `at` scheduling for timers (no permanent cron jobs)
- Always clean up timers after completion
- Never modify or delete existing issues without user consent
