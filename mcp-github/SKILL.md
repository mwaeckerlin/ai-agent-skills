# MCP GitHub Skill

Use this skill for secure GitHub MCP usage from sandboxed agents — list allowed operations, execute validated REST family tools, use GraphQL fallback safely, and troubleshoot auth/permission issues.

## Repository creation with Copilot

**Critical:** When creating new repositories that will use GitHub Copilot:

- **Problem:** Empty repositories (no base branch) cannot be used by Copilot cloud agents
- **Error:** "Copilot cloud agent can't start because the repository has no base branch"

**Solutions (choose one):**

1. **Preferred:** Pass the Copilot task/instructions **during repository creation** (not as a separate issue)
   - This ensures the repository is properly initialized with content
   - Avoids the empty repository problem entirely

2. **Alternative:** Initialize repository with content during creation
   - Enable "Add a README file" option
   - Or add license, .gitignore, or any initial commit

**Workflow:**
- When user requests new repository + Copilot work: combine into single repository creation with initial task
- Never create empty repository first, then try to assign Copilot via separate issue
- Repository must have at least one commit/branch for Copilot to work

## Copilot task status definitions

**CRITICAL:** Understand when Copilot tasks are actually finished:

- **FINISHED** = PR exists and is mergeable (`mergeable_state: "clean"`)
- **READY FOR REVIEW** = Same as FINISHED - draft status is irrelevant
- **NOT FINISHED** = No PR yet OR PR has conflicts/issues (`mergeable_state` != "clean")

**Status checking workflow:**
1. Check if PR exists for the task
2. If PR exists, check `mergeable_state` field
3. If `clean` = FINISHED (ignore `draft: true` and `merged: false`)
4. If conflicts/issues = NOT FINISHED

**Common mistakes to avoid:**
- ❌ Thinking `"draft": true` means "not finished"
- ❌ Thinking `"merged": false` means "not finished"
- ❌ Waiting for "ready for review" status changes
- ✅ Only check: PR exists + mergeable_state clean = FINISHED

**Examples:**
- Username placeholders like "mwaeckerlin" are samples - substitute your actual GitHub username/organization
- All repository and owner references should be adapted to your specific context

## Safe operating rules

This is a template for a MCP GitHub skill. The actual implementation should include detailed guidance on using the GitHub MCP service securely and effectively.