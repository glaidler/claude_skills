---
name: "plan-jira"
description: "Plan-first workflow: create a plan PR from a Jira ticket, iterate on feedback, then implement."
---

# Jira Plan Skill

This skill implements a three-phase workflow that bridges Jira tickets to code via plan PRs.

---

## Phase 1: Plan Creation

When no plan branch exists for the given Jira key, create everything from scratch.

### Step 1 — Retrieve Jira Ticket

1. Get the Atlassian cloud ID:
   - Call `getAccessibleAtlassianResources` to get the cloud ID
2. Fetch the Jira issue:
   - Call `getJiraIssue` with the cloud ID and issue key
3. Extract from the response:
   - **Title** (summary)
   - **Type** (Bug, Story, Task, etc.)
   - **Priority**
   - **Description**
   - **Acceptance criteria** (if present in description or custom fields)

### Step 2 — Determine Base Branch

Ask the user which branch to base the work on. Suggest the current branch or `main` as the default. Wait for user confirmation before proceeding.

### Step 3 — Create Branches

1. Fetch and checkout the base branch:
   ```
   git fetch origin
   git checkout <base-branch>
   git pull origin <base-branch>
   ```

2. Slugify the Jira title: lowercase, replace non-alphanumeric characters with hyphens, collapse multiple hyphens, trim leading/trailing hyphens, truncate to keep the total branch name reasonable (under 60 chars).
   - Example: "Fix Login Timeout on Mobile" → `fix-login-timeout-on-mobile`

3. Create the implementation branch:
   ```
   git checkout -b <JIRA_KEY>-<slugified-title>
   ```
   Example: `PROJ-123-fix-login-timeout-on-mobile`

4. Push the implementation branch so it exists on remote:
   ```
   git push -u origin <implementation-branch>
   ```

5. Create the plan branch from the implementation branch:
   ```
   git checkout -b <implementation-branch>__plan
   ```

### Step 4 — Explore the Codebase

Before writing the plan, understand the codebase:

1. Look at the project structure (key directories, config files, README)
2. Search for code relevant to the Jira ticket (related files, functions, modules)
3. Identify the areas that will need changes
4. Note any patterns, conventions, or testing approaches used in the project

Spend adequate time here — the quality of the plan depends on understanding the code.

### Step 5 — Write the Plan

Create the plan file at `.claude/plans/<JIRA_KEY>.md` using this template:

```markdown
# <JIRA_KEY>: <Jira Title>

## Jira Details
- **Type:** <issue type>
- **Priority:** <priority>
- **Description:** <description from Jira>

## Analysis
<Your analysis of the codebase and the problem. Reference specific files and code.>

## Approach
<High-level strategy for solving the problem. Explain WHY this approach was chosen.>

## Implementation Steps
1. <Step 1> — `path/to/file`
2. <Step 2> — `path/to/file`
...

## Files to Modify
- `path/to/file.ts` — <reason for change>
- `path/to/other.ts` — <reason for change>

## New Files
- `path/to/new-file.ts` — <purpose>

## Testing Strategy
- <How to verify the changes work>
- <Specific test cases to add or modify>

## Risks & Considerations
- <Edge cases>
- <Potential breaking changes>
- <Dependencies or coordination needed>
```

Fill in every section thoroughly. The plan should be detailed enough that someone could implement it without additional context.

### Step 6 — Commit & Push

```
git add .claude/plans/<JIRA_KEY>.md
git commit -m "plan: <JIRA_KEY> - <Jira Title>"
git push -u origin <plan-branch>
```

### Step 7 — Create Draft PR

Create a draft PR from the plan branch targeting the implementation branch:

```
gh pr create \
  --base <implementation-branch> \
  --head <plan-branch> \
  --title "📋 Plan: <JIRA_KEY> - <Jira Title>" \
  --body "$(cat <<'EOF'
## Plan for <JIRA_KEY>

This PR contains the implementation plan for [<JIRA_KEY>](<jira-url>).

### How to review
1. Read `.claude/plans/<JIRA_KEY>.md`
2. Leave comments on the plan file or on the PR
3. Approve when the plan looks good

### Next steps
Once approved, run `/jira-plan <JIRA_KEY> implement` to execute the plan.

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)" \
  --draft
```

### Step 8 — Output Result

Tell the user:
- The plan PR URL
- The implementation branch name
- How to proceed: review the PR, leave comments, then run `/jira-plan <JIRA_KEY>` to iterate or `/jira-plan <JIRA_KEY> implement` to implement

---

## Phase 2: Review & Adjust

When a plan branch and PR already exist, and `implement` was NOT passed.

### Step 1 — Find the Existing PR

```
gh pr list --head <plan-branch> --json number,url,state --jq '.[0]'
```

If no PR found, check if the branch exists and offer to recreate the PR.

### Step 2 — Read PR Feedback

Gather all feedback:

1. PR review comments (inline on files):
   ```
   gh api repos/{owner}/{repo}/pulls/<pr-number>/comments --jq '.[] | {path: .path, body: .body, line: .line, user: .user.login}'
   ```

2. PR conversation comments:
   ```
   gh pr view <pr-number> --comments --json comments --jq '.comments[] | {body: .body, user: .author.login}'
   ```

3. PR review status:
   ```
   gh pr view <pr-number> --json reviews --jq '.reviews[] | {state: .state, body: .body, user: .author.login}'
   ```

### Step 3 — Checkout Plan Branch

```
git fetch origin
git checkout <plan-branch>
git pull origin <plan-branch>
```

### Step 4 — Update the Plan

1. Read the current plan from `.claude/plans/<JIRA_KEY>.md`
2. Analyze the feedback:
   - Summarize each piece of feedback for the user
   - Suggest changes to address each comment
3. Ask the user if the proposed changes look good
4. Update `.claude/plans/<JIRA_KEY>.md` with the changes

### Step 5 — Commit & Push

```
git add .claude/plans/<JIRA_KEY>.md
git commit -m "plan: update <JIRA_KEY> based on review feedback"
git push origin <plan-branch>
```

### Step 6 — Notify User

Tell the user:
- What changes were made to the plan
- The PR URL for continued review
- Remind them to run `/jira-plan <JIRA_KEY> implement` when the plan is approved

---

## Phase 3: Implement

When `implement` is explicitly passed as the second argument. This is the destructive phase that writes real code.

### Step 1 — Verify Plan Status

1. Find the plan PR:
   ```
   gh pr list --head <plan-branch> --json number,url,state,reviews --jq '.[0]'
   ```

2. Check if the PR has been approved or merged:
   - If **approved** → proceed
   - If **merged** → proceed
   - If **neither** → warn the user that the plan hasn't been approved yet and ask if they want to proceed anyway. Do NOT proceed without explicit confirmation.

### Step 2 — Switch to Implementation Branch

```
git fetch origin
git checkout <implementation-branch>
git pull origin <implementation-branch>
```

If the plan PR was merged, the plan file should already be on this branch. If not, read it from the plan branch:
```
git show <plan-branch>:.claude/plans/<JIRA_KEY>.md
```

### Step 3 — Read the Plan

Read `.claude/plans/<JIRA_KEY>.md` and parse the implementation steps.

Present the plan summary to the user and ask for confirmation before starting implementation.

### Step 4 — Execute the Plan

Work through each implementation step from the plan:

1. For each step:
   - Tell the user which step you're working on
   - Make the code changes described in the plan
   - Verify the changes compile/lint if applicable

2. Follow the testing strategy from the plan:
   - Run existing tests to ensure nothing is broken
   - Add new tests as specified in the plan

3. Commit logically — group related changes into coherent commits rather than one giant commit.

### Step 5 — Final Review

After implementation is complete:

1. Run the full test suite if one exists
2. Show the user a summary of all changes made
3. Ask the user to review the changes
4. Offer to create a PR from the implementation branch to the base branch:
   ```
   gh pr create \
     --base <base-branch> \
     --head <implementation-branch> \
     --title "<JIRA_KEY>: <Jira Title>" \
     --body "$(cat <<'EOF'
   ## Summary
   <brief summary of changes>

   ## Jira
   [<JIRA_KEY>](<jira-url>)

   ## Plan
   See `.claude/plans/<JIRA_KEY>.md` for the full implementation plan.

   ## Changes
   <list of key changes>

   ## Test plan
   <how the changes were tested>

   ---
   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

---

## Branch Naming Reference

| Branch | Pattern | Example |
|--------|---------|---------|
| Implementation | `<JIRA_KEY>-<slugified-title>` | `PROJ-123-fix-login-timeout` |
| Plan | `<implementation-branch>__plan` | `PROJ-123-fix-login-timeout__plan` |

## Slugification Rules

1. Take the Jira issue summary (title)
2. Convert to lowercase
3. Replace any character that isn't `a-z`, `0-9`, or `-` with a hyphen
4. Collapse consecutive hyphens into one
5. Remove leading and trailing hyphens
6. Truncate so the full branch name (including Jira key prefix) stays under 60 characters
