---
description: "Plan and implement Jira tickets via plan-first PR workflow"
allowed-tools: "Bash, Read, Write, Edit, Glob, Grep, WebFetch, mcp__claude_ai_Atlassian__getAccessibleAtlassianResources, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql"
---

# /jira-plan

You are executing the `jira-plan` skill. This command bridges Jira tickets to implementation via a plan-first PR workflow.

## Gather Context

First, collect the current state:

1. Parse the arguments: `$ARGUMENTS`
   - Extract the Jira issue key (e.g., `PROJ-123`) — this is the first argument and is **required**
   - Check if `implement` was passed as a second argument
   - If no Jira key is provided, ask the user for one and stop

2. Get current git state:
   - Current branch: run `git branch --show-current`
   - Git status: run `git status --porcelain`
   - If there are uncommitted changes, warn the user and ask how to proceed (stash, commit, or abort)

3. Detect the current phase by checking if a plan branch already exists:
   - Run: `git branch -a --list "*<JIRA_KEY>*__plan*"`
   - If **no plan branch exists** → proceed with **Phase 1: Plan Creation**
   - If **plan branch exists** and `implement` arg was **NOT** passed → proceed with **Phase 2: Review & Adjust**
   - If **plan branch exists** and `implement` arg **WAS** passed → proceed with **Phase 3: Implement**

Now load and follow the full instructions in `skills/plan-jira/SKILL.md` for the detected phase.
