---
name: "engage-blueprint"
description: "Plan-first workflow: create a plan in Confluence from a Jira ticket, iterate on feedback, then implement."
---

# Jira Plan Skill

This skill implements a three-phase workflow that bridges Jira tickets to code. Plans are reviewed in Confluence, workflow is managed via Jira ticket status, and code is delivered via pull requests.

**Communication rules**: The user may not be a developer. Never use git terminology (commit, push, stash, checkout, branch, merge, rebase, pull, PR) in messages to the user. Execute all git and CLI commands silently. Only surface results, progress, and errors in plain language. If a command fails, translate the error into a friendly message rather than showing raw output.

---

## Phase 1: Plan Creation

When the Jira ticket is in an early-stage status (To Do, Open, Backlog, or similar).

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
   - **Current status**

Tell the user: "I've loaded the details for <JIRA_KEY>: <Title>. Let me explore the codebase and create a plan."

### Step 2 — Determine Starting Point

The work will be based on the `main` branch by default.

- If the user is already on `main`, proceed automatically. Tell them: "I'll base this work on the main codebase."
- If the user is on a different branch, tell them: "You're currently working from a place called `<branch-name>`. Normally I start from the main codebase. Should I use **main** (recommended), or start from where you are now?" Wait for the user to choose.

Do NOT use terms like "base branch" — say "starting point" if needed.

### Step 3 — Set Up Working Area

Create the branch needed for implementation. The user does not need to understand branches — communicate as "setting up a working area for this ticket." Execute git commands silently and only surface errors in friendly language.

1. Fetch and checkout the starting point:
   ```
   git fetch origin
   git checkout <starting-branch>
   git pull origin <starting-branch>
   ```

2. Slugify the Jira title: lowercase, replace non-alphanumeric characters with hyphens, collapse multiple hyphens, trim leading/trailing hyphens, truncate to keep the total branch name under 60 chars.
   - Example: "Fix Login Timeout on Mobile" → `fix-login-timeout-on-mobile`

3. Create the implementation branch:
   ```
   git checkout -b <JIRA_KEY>-<slugified-title>
   ```

4. Push the implementation branch so it exists on remote:
   ```
   git push -u origin <implementation-branch>
   ```

> **Error translation guide** — if any git command fails, do NOT show the raw error. Translate it:
> - "fatal: A branch named 'X' already exists" → "It looks like work on this ticket was already started. Let me check the current state."
> - "error: failed to push" → "I wasn't able to upload the working area. This might be a permissions issue — please check with your team."
> - Any other error → "Something went wrong while setting up. Here's a summary: [one-sentence plain description]. You may want to ask a developer for help."

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

### Step 6 — Save the Plan Locally

Save the plan file silently:

```
git add .claude/plans/<JIRA_KEY>.md
git commit -m "plan: <JIRA_KEY> - <Jira Title>"
git push origin <implementation-branch>
```

Do not mention commits or pushes to the user.

### Step 7 — Publish Plan to Confluence

1. **Determine the Confluence space**:
   - Check if a space preference has been stored previously in `.claude/plans/.confluence-config.json`
   - If no preference exists, call `getConfluenceSpaces` to list available spaces
   - Ask the user: "Where should I publish the plan for review? Here are the available spaces:" and list the space names
   - Store the user's choice in `.claude/plans/.confluence-config.json` as `{"spaceKey": "<KEY>"}` for future runs

2. **Create the Confluence page**:
   - Call `createConfluencePage` with:
     - Space key from config
     - Title: `Plan: <JIRA_KEY> - <Jira Title>`
     - Body: the contents of the plan file (converted to Confluence-compatible format)
   - Store the page ID in `.claude/plans/.confluence-config.json` under a key for this Jira issue, e.g., `{"spaceKey": "<KEY>", "pages": {"<JIRA_KEY>": "<PAGE_ID>"}}`

3. **Link the plan on the Jira ticket**:
   - Call `addCommentToJiraIssue` to post a comment on the ticket:
     > "A plan has been created for this ticket. Please review it here: [Plan: <JIRA_KEY> - <Jira Title>](<confluence-page-url>)"

### Step 8 — Transition the Jira Ticket

1. Call `getTransitionsForJiraIssue` to get available transitions
2. Look for a transition that moves the ticket to a review-like status (e.g., "In Review", "Planning", "In Progress")
3. If a suitable transition is found, call `transitionJiraIssue` to move the ticket
4. If no obvious transition exists, tell the user: "I've published the plan but I'm not sure which status to move the ticket to. Could you update the ticket status in Jira to indicate it's ready for review?"

### Step 9 — Tell the User What Happened

Tell the user in friendly language:

- "Your plan is ready for review!"
- Provide the Confluence page link: "You can read and comment on it here: [link]"
- "Here's what to do next:"
  1. "Open the link above and read through the plan"
  2. "Leave comments on anything you'd like changed"
  3. "Once you're happy with the plan, move the Jira ticket to an approved status (like 'Ready for Dev' or 'Approved')"
  4. "Then come back and run `/engage-blueprint <JIRA_KEY> implement` to start building"
- "If you get feedback and want me to update the plan, just run `/engage-blueprint <JIRA_KEY>` again and I'll pick up the comments."

### Step 10 — Restore Saved Work

If the user's work was saved aside at the start (via git stash), restore it now:
```
git stash list | grep "engage-blueprint: saved before <JIRA_KEY>"
```
If found, run `git stash pop` and tell the user: "I've restored the work you had in progress before we started."

---

## Phase 2: Review & Adjust

When the Jira ticket is in a review/planning status and `implement` was NOT passed.

### Step 1 — Find the Existing Plan

1. Read the Confluence config from `.claude/plans/.confluence-config.json` to get the page ID for this Jira issue
2. If no config exists, search Confluence for the plan page:
   - Call `getConfluenceSpaces` then search for a page titled `Plan: <JIRA_KEY>*`
3. If no plan page is found, tell the user: "I couldn't find an existing plan for this ticket. Would you like me to create one? Run `/engage-blueprint <JIRA_KEY>` on a ticket in 'To Do' status to start fresh."

### Step 2 — Read Feedback

Gather all feedback from Confluence:

1. Call `getConfluencePageFooterComments` with the page ID to get reviewer comments
2. Call `getConfluencePage` to read the current plan content

### Step 3 — Load the Latest Plan

Silently ensure you have the latest version:

```
git fetch origin
git checkout <implementation-branch>
git pull origin <implementation-branch>
```

Tell the user: "I'm loading the latest version of your plan." Do not mention git operations.

### Step 4 — Update the Plan

1. Read the current plan from `.claude/plans/<JIRA_KEY>.md`
2. Analyze the feedback:
   - Summarize each piece of feedback for the user in plain language
   - Suggest changes to address each comment
3. Ask the user if the proposed changes look good
4. Update `.claude/plans/<JIRA_KEY>.md` with the changes

### Step 5 — Save and Upload Updates

Save changes silently:

```
git add .claude/plans/<JIRA_KEY>.md
git commit -m "plan: update <JIRA_KEY> based on review feedback"
git push origin <implementation-branch>
```

Then update the Confluence page:
- Call `updateConfluencePage` with the page ID and the updated plan content

Optionally reply to specific comments on the Confluence page using `createConfluenceFooterComment` to acknowledge feedback was addressed.

### Step 6 — Tell the User

Tell the user:
- "I've updated your plan based on the review feedback."
- Summarize the changes in bullet points
- Provide the Confluence page link: "You can see the updated plan here: [link]"
- "Once the reviewer is happy and moves the ticket to an approved status, run `/engage-blueprint <JIRA_KEY> implement` to start building."

### Step 7 — Restore Saved Work

If the user's work was saved aside at the start (via git stash), restore it now:
```
git stash list | grep "engage-blueprint: saved before <JIRA_KEY>"
```
If found, run `git stash pop` and tell the user: "I've restored the work you had in progress before we started."

---

## Phase 3: Implement

When the Jira ticket is in an approved status and `implement` was explicitly passed. This phase writes real code.

### Step 1 — Verify the Ticket is Approved

1. Fetch the Jira issue using `getJiraIssue` and check its current status
2. Check whether the status indicates approval:
   - If status is **"Approved"**, **"Ready for Dev"**, **"Selected for Development"**, or similar → proceed to Step 2
   - If status is **anything else** → **STOP**. Do NOT offer to proceed. Tell the user: "This ticket hasn't been approved yet. Before I can start building, someone needs to review the plan and move the ticket to an approved status in Jira. You can find the plan here: [Confluence page link]. Once the ticket is approved, come back and run this command again."

This is a hard gate. Under no circumstances should implementation proceed without the ticket being in an approved status.

**Escape valve**: If the user says they are the only person on the project and cannot get someone else to approve, tell them: "You can move the ticket to the approved status yourself in Jira, and then run this command again."

### Step 2 — Prepare to Build

Tell the user: "The plan is approved! I'm setting things up to start building."

Silently switch to the implementation branch:
```
git fetch origin
git checkout <implementation-branch>
git pull origin <implementation-branch>
```

If the implementation branch doesn't exist, create it from the starting point (default `main`):
```
git fetch origin
git checkout main
git pull origin main
git checkout -b <JIRA_KEY>-<slugified-title>
git push -u origin <implementation-branch>
```

### Step 3 — Read the Plan

Read `.claude/plans/<JIRA_KEY>.md` and parse the implementation steps.

If the plan file doesn't exist locally, fetch it from Confluence:
- Call `getConfluencePage` using the page ID from `.claude/plans/.confluence-config.json`
- Save the content to `.claude/plans/<JIRA_KEY>.md`

Present the plan summary to the user in plain language and ask for confirmation: "Here's what I'm going to build: [summary]. Ready to go?"

### Step 4 — Execute the Plan

Work through each implementation step from the plan:

1. For each step:
   - Use plain language when describing progress. Say "I'm working on [description of what the step accomplishes]" rather than "I'm modifying [filename]". Only mention filenames if the user has demonstrated technical comfort.
   - Make the code changes described in the plan
   - Verify the changes compile/lint if applicable

2. Follow the testing strategy from the plan:
   - Run existing tests to ensure nothing is broken
   - Add new tests as specified in the plan

3. Save work logically — group related changes into coherent saves rather than one giant save. Execute git commands silently:
   ```
   git add <relevant-files>
   git commit -m "<descriptive message>"
   ```

### Step 5 — Final Review

After implementation is complete:

1. Run the full test suite if one exists. Tell the user: "I'm running tests to make sure everything works."
2. Show the user a plain-language summary of what was built, organized by what changed from the user's perspective (not file-by-file).
3. Ask: "Would you like me to submit this for code review?"
4. If yes, create the PR:
   ```
   gh pr create \
     --base <starting-branch> \
     --head <implementation-branch> \
     --title "<JIRA_KEY>: <Jira Title>" \
     --body "$(cat <<'EOF'
   ## Summary
   <brief summary of changes>

   ## Jira
   [<JIRA_KEY>](<jira-url>)

   ## Plan
   See the [implementation plan](<confluence-page-url>) for full details.

   ## Changes
   <list of key changes>

   ## Test plan
   <how the changes were tested>

   ---
   Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```
   Tell the user: "Done! Your code has been submitted for review: [PR URL]"

5. Update the Jira ticket:
   - Call `addCommentToJiraIssue` to post: "Implementation complete. Code review: [PR URL]"
   - Optionally call `transitionJiraIssue` to move the ticket to an "In Code Review" or "In Progress" status if available

### Step 6 — Restore Saved Work

If the user's work was saved aside at the start (via git stash), restore it now:
```
git stash list | grep "engage-blueprint: saved before <JIRA_KEY>"
```
If found, run `git stash pop` and tell the user: "I've restored the work you had in progress before we started."

---

## Branch Naming Reference

| Branch | Pattern | Example |
|--------|---------|---------|
| Implementation | `<JIRA_KEY>-<slugified-title>` | `PROJ-123-fix-login-timeout` |

## Slugification Rules

1. Take the Jira issue summary (title)
2. Convert to lowercase
3. Replace any character that isn't `a-z`, `0-9`, or `-` with a hyphen
4. Collapse consecutive hyphens into one
5. Remove leading and trailing hyphens
6. Truncate so the full branch name (including Jira key prefix) stays under 60 characters

## Confluence Config File

The file `.claude/plans/.confluence-config.json` stores persistent configuration:

```json
{
  "spaceKey": "TEAM",
  "pages": {
    "PROJ-123": "confluence-page-id-here",
    "PROJ-456": "confluence-page-id-here"
  }
}
```

This file is created automatically on first run when the user selects a Confluence space. It allows subsequent runs to find existing plan pages without searching.
