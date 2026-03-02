---
description: "Plan and implement Jira tickets via plan-first workflow"
allowed-tools: "Bash, Read, Write, Edit, Glob, Grep, WebFetch, mcp__claude_ai_Atlassian__getAccessibleAtlassianResources, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql, mcp__claude_ai_Atlassian__getTransitionsForJiraIssue, mcp__claude_ai_Atlassian__transitionJiraIssue, mcp__claude_ai_Atlassian__addCommentToJiraIssue, mcp__claude_ai_Atlassian__createConfluencePage, mcp__claude_ai_Atlassian__updateConfluencePage, mcp__claude_ai_Atlassian__getConfluencePage, mcp__claude_ai_Atlassian__getConfluencePageFooterComments, mcp__claude_ai_Atlassian__createConfluenceFooterComment, mcp__claude_ai_Atlassian__getConfluenceSpaces, mcp__claude_ai_Atlassian__atlassianUserInfo"
---

# /engage-blueprint

You are executing the `engage-blueprint` skill. This command bridges Jira tickets to implementation via a plan-first workflow using Confluence for plan review and Jira for workflow management.

**Important**: The user may not be a developer. All communication must use plain, non-technical language. Never use git terminology (commit, push, stash, checkout, branch, merge, rebase, pull) in messages to the user unless they explicitly ask for technical details. Execute git commands silently and only surface results or errors in friendly language.

## Phase 0 — Verify Setup

Before doing anything else, silently verify that the required tools are available. If any check fails, stop immediately with a clear, non-technical message.

1. **Git** — Run `git --version`
   - On failure: "It looks like Git isn't installed on this machine. Please ask your team's developer to set it up."

2. **GitHub CLI** — Run `gh --version`
   - On failure: "The GitHub command-line tool isn't installed. Please ask your team's developer to install it from https://cli.github.com/."
   - Then run `gh auth status` to check authentication.
   - On auth failure: "The GitHub tool is installed but not signed in. Please ask your team's developer to run `gh auth login` in a terminal."

3. **Git repository** — Run `git rev-parse --is-inside-work-tree`
   - On failure: "This folder isn't set up as a code project. Please make sure you're in the right folder before running this command."

4. **Atlassian connection** — Call `getAccessibleAtlassianResources`
   - On any failure: "I can't connect to Jira and Confluence. The Atlassian integration may not be set up yet. Please check that the Atlassian MCP server is configured in your Claude Code settings."

Only proceed if all four checks pass.

## Gather Context

1. Parse the arguments: `$ARGUMENTS`
   - Extract the Jira issue key (e.g., `PROJ-123`) — this is the first argument and is **required**
   - Check if `implement` was passed as a second argument
   - If no Jira key is provided, ask the user for one and stop

2. Get current git state (silently):
   - Current branch: run `git branch --show-current`
   - Git status: run `git status --porcelain`
   - If there are uncommitted changes, tell the user: "You have unsaved work in this folder. I need a clean starting point before I can continue. Should I **save your current work aside** (I'll restore it when we're done), or would you rather **stop here** so you can handle it yourself?"
     - If save aside: run `git stash push -m "engage-blueprint: saved before <JIRA_KEY>"`
     - If stop: end the command with a friendly message

3. Detect the current phase using the Jira ticket status:
   - Fetch the Jira issue using `getJiraIssue` and read its current status
   - Map the status to a phase:
     - **Status is "To Do", "Open", "Backlog", or similar early-stage status** → proceed with **Phase 1: Plan Creation**
     - **Status is "In Review", "Planning", "In Progress" (and `implement` was NOT passed)** → proceed with **Phase 2: Review & Adjust**
     - **Status is "Approved", "Ready for Dev", "Selected for Development", or similar approved status** AND `implement` was passed → proceed with **Phase 3: Implement**
     - **Status is an approved status** but `implement` was NOT passed → tell the user: "This ticket is approved and ready to build. Run `/engage-blueprint <JIRA_KEY> implement` to start implementation."
     - **If the status doesn't clearly match any phase** → tell the user the current status and ask which phase they'd like to run

Now load and follow the full instructions in `skills/engage-blueprint/SKILL.md` for the detected phase.
