---
name: "engage-blueprint"
description: "Plan-first workflow: create a plan in Confluence from a Jira ticket, iterate on feedback, then implement."
---

# Jira Plan Skill

This skill implements a three-phase workflow that bridges Jira tickets to code. Plans are reviewed in Confluence, workflow is managed via Jira ticket status, and code is delivered via pull requests.

**Communication rules**: The user may not be a developer. Never use git terminology (commit, push, stash, checkout, branch, merge, rebase, pull, PR) in messages to the user. Execute all git and CLI commands silently. Only surface results, progress, and errors in plain language. If a command fails, translate the error into a friendly message rather than showing raw output.

---

## Pre-Phase Check — Verify Git

Before starting any phase, silently verify that Git is available. If any check fails, stop immediately with a clear, non-technical message.

1. **Git** — Run `git --version`
   - On failure: "It looks like Git isn't installed on this machine. Please ask your team's developer to set it up."

2. **Git repository** — Run `git rev-parse --is-inside-work-tree`
   - On failure: "This folder isn't set up as a code project. Please make sure you're in the right folder before running this command."

Only proceed if both checks pass.

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

### Step 2 — Explore the Codebase

Before writing the plan, understand the codebase:

1. Look at the project structure (key directories, config files, README)
2. Search for code relevant to the Jira ticket (related files, functions, modules)
3. Identify the areas that will need changes
4. Note any patterns, conventions, or testing approaches used in the project

Spend adequate time here — the quality of the plan depends on understanding the code.

### Step 3 — Write the Plan

Create the plan file at `.claude/plans/<JIRA_KEY>.md` using this template. The template has two distinct halves: a **business summary** at the top for stakeholders and reviewers (written in plain, non-technical language), and a **technical plan** below the divider for developers.

```markdown
# <JIRA_KEY>: <Jira Title>

## Summary

<2-3 sentence plain-language explanation of what this change does and why it's needed. No technical jargon. Write this for someone who uses the product but doesn't read code.>

## What Changes for Users

- <Visible change 1 — describe from the end-user's perspective>
- <Visible change 2>
- <If there are no user-visible changes, say "This is an under-the-hood improvement. Users won't see any difference, but it [improves performance / fixes a rare issue / etc.].">

## Decisions Needed

> **Reviewers**: please reply with your answers by number (e.g. "1. Yes, 2. Option A") so everything fits in one comment.

1. <Question or decision 1 that needs stakeholder input>
2. <Question or decision 2>
<If no decisions are needed, replace the numbered list with "No open questions — this one is straightforward.">

## UI Mockup

<Include this section ONLY if the ticket involves user-visible UI changes. Delete it entirely for backend-only or non-visual work.>

<ASCII wireframe showing the before and after. Use box-drawing characters to represent the layout. Label interactive elements. Example:>

```
BEFORE:
┌──────────────────────────────┐
│  Header            [Log in]  │
└──────────────────────────────┘

AFTER:
┌──────────────────────────────┐
│  Header    [Sign up] [Log in]│
└──────────────────────────────┘
```

<Show only the parts of the UI that change. Keep it simple — this is a sketch, not a spec.>

## Risks & Impact

- <Plain-language risk 1, e.g. "Users will be logged out when we deploy this">
- <Plain-language risk 2, e.g. "This changes how notifications work, so we should let the support team know">
- <If low risk, say "Low risk — this change is isolated and doesn't affect other parts of the product.">

---

*Everything below this line is technical detail for developers.*

---

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

## Technical Risks
- <Edge cases>
- <Potential breaking changes>
- <Dependencies or coordination needed>
```

Fill in every section thoroughly. The business summary sections should be understandable by anyone on the team. The technical sections should be detailed enough that a developer could implement the plan without additional context.

### Step 4 — Save the Plan Locally

Save the plan file to `.claude/plans/<JIRA_KEY>.md`. No git operations are needed at this stage — the plan is saved locally and will be published to Confluence in the next step.

### Step 5 — Publish Plan to Confluence

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

3. **Post the plan summary and questions on the Jira ticket**:
   - Call `addCommentToJiraIssue` to post a comment that includes the business-facing sections from the plan. Use this format:

     > ## Plan ready for review
     >
     > **Summary:** <Summary section from the plan>
     >
     > **What changes for users:**
     > - <bullet points from the "What Changes for Users" section>
     >
     > **Risks & impact:**
     > - <bullet points from the "Risks & Impact" section>
     >
     > ### Decisions needed
     > 1. <question 1>
     > 2. <question 2>
     >
     > **Reply with your answers by number (e.g. "1. Yes, 2. Option A") so everything fits in one comment.**
     >
     > Full plan details: [Plan: <JIRA_KEY> - <Jira Title>](<confluence-page-url>)

   - This lets stakeholders review and respond directly in Jira without needing to visit Confluence. The Confluence link is still included for anyone who wants the full technical detail.

### Step 6 — Transition the Jira Ticket

1. Call `getTransitionsForJiraIssue` to get available transitions
2. Look for a transition that moves the ticket to a review-like status (e.g., "In Review", "Planning", "In Progress")
3. If a suitable transition is found, call `transitionJiraIssue` to move the ticket
4. If no obvious transition exists, tell the user: "I've published the plan but I'm not sure which status to move the ticket to. Could you update the ticket status in Jira to indicate it's ready for review?"

### Step 7 — Tell the User What Happened

Tell the user in friendly language:

- "Your plan is ready for review!"
- Provide the Confluence page link: "You can read and comment on it here: [link]"
- "Here's what to do next:"
  1. "Open the link above and read through the plan"
  2. "Leave comments on anything you'd like changed"
  3. "Once you're happy with the plan, move the Jira ticket to an approved status (like 'Ready for Dev' or 'Approved')"
  4. "Then come back and run `/engage-blueprint <JIRA_KEY> implement` to start building"
- "If you get feedback and want me to update the plan, just run `/engage-blueprint <JIRA_KEY>` again and I'll pick up the comments."

---

## Phase 2: Review & Adjust

When the Jira ticket is in a review/planning status and `implement` was NOT passed.

### Step 1 — Find the Existing Plan

1. Read the Confluence config from `.claude/plans/.confluence-config.json` to get the page ID for this Jira issue
2. If no config exists, search Confluence for the plan page:
   - Call `getConfluenceSpaces` then search for a page titled `Plan: <JIRA_KEY>*`
3. If no plan page is found, tell the user: "I couldn't find an existing plan for this ticket. Would you like me to create one? Run `/engage-blueprint <JIRA_KEY>` on a ticket in 'To Do' status to start fresh."

### Step 2 — Read Feedback

Gather feedback from both Jira and Confluence:

1. **Jira comments** — Call `getJiraIssue` with the cloud ID and issue key, and read any comments on the ticket. Look for replies to the "Plan ready for review" comment that contain stakeholder answers to the decisions needed.
2. **Confluence comments** — Call `getConfluencePageFooterComments` with the page ID to get reviewer comments left on the plan page.
3. Call `getConfluencePage` to read the current plan content.

Combine feedback from both sources. If the same question was answered in both places, prefer the most recent response.

### Step 3 — Update the Plan

1. Read the current plan from `.claude/plans/<JIRA_KEY>.md` (or fetch from Confluence if the local file is missing)
2. Analyze the feedback:
   - Summarize each piece of feedback for the user in plain language
   - Suggest changes to address each comment
3. Ask the user if the proposed changes look good
4. Update `.claude/plans/<JIRA_KEY>.md` with the changes

### Step 4 — Save and Upload Updates

Save the updated plan to `.claude/plans/<JIRA_KEY>.md` locally, then update Confluence:
- Call `updateConfluencePage` with the page ID and the updated plan content

Optionally reply to specific comments on the Confluence page using `createConfluenceFooterComment` to acknowledge feedback was addressed.

### Step 5 — Tell the User

Tell the user:
- "I've updated your plan based on the review feedback."
- Summarize the changes in bullet points
- Provide the Confluence page link: "You can see the updated plan here: [link]"
- "Once the reviewer is happy and moves the ticket to an approved status, run `/engage-blueprint <JIRA_KEY> implement` to start building."

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

### Step 2 — Check for Unaddressed Feedback

Before building, check whether any comments were left after the plan was last updated:

1. Read the Confluence page using `getConfluencePage` — note the page's `version.when` (last modified timestamp).
2. Read Confluence footer comments using `getConfluencePageFooterComments`.
3. Read Jira comments from the ticket using `getJiraIssue`.
4. Compare timestamps: if any comments from either source are **newer** than the plan's last modified timestamp, they may not have been incorporated.

If unaddressed comments are found:
- List them for the user in plain language: "I found feedback that came in after the plan was last updated:"
  - 1. "<comment summary>" — <author>
  - 2. "<comment summary>" — <author>
- Ask: "Would you like me to **update the plan** to address these before building, or **go ahead** with the current plan?"
  - If update: run the Phase 2 review flow (Steps 2–5) and then return here.
  - If go ahead: proceed to Step 3.

If no unaddressed comments are found, proceed to Step 3.

### Step 3 — Prepare to Build

Tell the user: "The plan is approved! I'm setting things up to start building."

1. **Check for unsaved work** (silently):
   - Run `git status --porcelain`
   - If there are uncommitted changes, tell the user: "You have unsaved work in this folder. I need a clean starting point before I can build. Should I **save your current work aside** (I'll restore it when we're done), or would you rather **stop here** so you can handle it yourself?"
     - If save aside: run `git stash push -m "engage-blueprint: saved before <JIRA_KEY>"`
     - If stop: end the command with a friendly message

2. **Create the implementation branch** (silently):
   - Slugify the Jira title: lowercase, replace non-alphanumeric characters with hyphens, collapse multiple hyphens, trim leading/trailing hyphens, truncate to keep the total branch name under 60 chars.
     - Example: "Fix Login Timeout on Mobile" → `fix-login-timeout-on-mobile`
   - Create and push the branch:
     ```
     git fetch origin
     git checkout main
     git pull origin main
     git checkout -b <JIRA_KEY>-<slugified-title>
     git push -u origin <implementation-branch>
     ```
   - If the branch already exists, just switch to it:
     ```
     git fetch origin
     git checkout <implementation-branch>
     git pull origin <implementation-branch>
     ```

> **Error translation guide** — if any git command fails, do NOT show the raw error. Translate it:
> - "fatal: A branch named 'X' already exists" → "It looks like work on this ticket was already started. Let me pick up where we left off."
> - "error: failed to push" → "I wasn't able to upload the working area. This might be a permissions issue — please check with your team."
> - Any other error → "Something went wrong while setting up. Here's a summary: [one-sentence plain description]. You may want to ask a developer for help."

### Step 4 — Read the Plan

Read `.claude/plans/<JIRA_KEY>.md` and parse the implementation steps.

If the plan file doesn't exist locally, fetch it from Confluence:
- Call `getConfluencePage` using the page ID from `.claude/plans/.confluence-config.json`
- Save the content to `.claude/plans/<JIRA_KEY>.md`

Present the plan summary to the user in plain language and ask for confirmation: "Here's what I'm going to build: [summary]. Ready to go?"

### Step 5 — Execute the Plan

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

### Step 6 — Final Review

After implementation is complete:

1. Run the full test suite if one exists. Tell the user: "I'm running tests to make sure everything works."
2. Show the user a plain-language summary of what was built, organized by what changed from the user's perspective (not file-by-file).
3. Ask: "Would you like me to submit this for code review?"
4. If yes, first verify the GitHub CLI is available:
   - Run `gh --version`. On failure: "The GitHub command-line tool isn't installed. I've finished building everything, but I can't submit it for review automatically. Please ask your team's developer to install it from https://cli.github.com/ and then run this command again."
   - Run `gh auth status`. On failure: "The GitHub tool is installed but not signed in. Please ask your team's developer to run `gh auth login` in a terminal, and then run this command again."
   - If either check fails, stop here — do NOT attempt to create the PR.
5. Create the PR:
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

6. Update the Jira ticket:
   - Call `addCommentToJiraIssue` to post: "Implementation complete. Code review: [PR URL]"
   - Optionally call `transitionJiraIssue` to move the ticket to an "In Code Review" or "In Progress" status if available

### Step 7 — Restore Saved Work

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
