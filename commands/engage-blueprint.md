---
description: "Plan and implement Jira tickets via plan-first workflow"
allowed-tools: "Bash, Read, Write, Edit, Glob, Grep, WebFetch, mcp__claude_ai_Atlassian__getAccessibleAtlassianResources, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql, mcp__claude_ai_Atlassian__getTransitionsForJiraIssue, mcp__claude_ai_Atlassian__transitionJiraIssue, mcp__claude_ai_Atlassian__addCommentToJiraIssue, mcp__claude_ai_Atlassian__createConfluencePage, mcp__claude_ai_Atlassian__updateConfluencePage, mcp__claude_ai_Atlassian__getConfluencePage, mcp__claude_ai_Atlassian__getConfluencePageFooterComments, mcp__claude_ai_Atlassian__createConfluenceFooterComment, mcp__claude_ai_Atlassian__getConfluenceSpaces, mcp__claude_ai_Atlassian__atlassianUserInfo"
---

# /engage-blueprint

You are executing the `engage-blueprint` skill. This command bridges Jira tickets to implementation via a plan-first workflow using Confluence for plan review and Jira for workflow management.

**Important**: The user may not be a developer. All communication must use plain, non-technical language. Never use git terminology (commit, push, stash, checkout, branch, merge, rebase, pull) in messages to the user unless they explicitly ask for technical details. Execute git commands silently and only surface results or errors in friendly language.

## Phase 0 — Verify Setup

Before doing anything else, silently verify that the required tools are available. If any check fails, stop immediately with a clear, non-technical message.

1. **Atlassian connection** — Call `getAccessibleAtlassianResources`
   - On any failure: "I can't connect to Jira and Confluence. The Atlassian integration may not be set up yet. Please check that the Atlassian MCP server is configured in your Claude Code settings."

2. **Azure CLI** — Run `az --version`
   - On any failure: "The Azure command-line tool isn't installed. I need it to upload UI previews. Please ask your team to install it from https://learn.microsoft.com/en-us/cli/azure/install-azure-cli"

3. **Azure authentication** — Run `az account show`
   - On any failure: "Azure CLI is installed but not signed in. Please run `az login` in a terminal first, then try again."

Only proceed if all checks pass.

## Gather Context

1. Parse the arguments: `$ARGUMENTS`
   - Extract the Jira issue key (e.g., `PROJ-123`) — this is the first argument and is **required**
   - Check if `implement` was passed as a second argument
   - If no Jira key is provided, ask the user for one and stop

2. Detect the current phase using the Jira ticket status:
   - Fetch the Jira issue using `getJiraIssue` and read its current status
   - Map the status to a phase:
     - **Status is "To Do", "Open", "Backlog", or similar early-stage status** → proceed with **Phase 1: Plan Creation**
     - **Status is "In Review", "Planning", "In Progress" (and `implement` was NOT passed)** → proceed with **Phase 2: Review & Adjust**
     - **Status is "Approved", "Ready for Dev", "Selected for Development", or similar approved status** AND `implement` was passed → proceed with **Phase 3: Implement**
     - **Status is an approved status** but `implement` was NOT passed → tell the user: "This ticket is approved and ready to build. Run `/engage-blueprint <JIRA_KEY> implement` to start implementation."
     - **If the status doesn't clearly match any phase** → tell the user the current status and ask which phase they'd like to run

Now load and follow the full instructions in `skills/engage-blueprint/SKILL.md` for the detected phase.
