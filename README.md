# Engage Blueprint

A Claude Code plugin that bridges Jira tickets to code through a plan-first workflow. Plans are reviewed in Confluence, workflow is managed via Jira ticket status, and code is delivered via pull requests.

## Install

Add to your project's `.claude/settings.json`:

```json
{
  "permissions": {
    "atlassian-blueprint@atlassian-claude-plugins": true
  }
}
```

Or install via the Claude Code CLI:

```
claude install github:glaidler/claude-skills
```

## Usage

```
/engage-blueprint <JIRA_KEY>
/engage-blueprint <JIRA_KEY> implement
```

## How It Works

The workflow has three phases:

### Phase 1 — Plan

Run `/atlassian-blueprint PROJ-123` on a ticket in "To Do" status. The skill:

1. Reads the Jira ticket details
2. Explores the codebase to understand what needs to change
3. Writes a plan with a business summary and technical implementation steps
4. Publishes the plan to Confluence and posts a summary on the Jira ticket
5. Moves the ticket to a review status

### Phase 2 — Review

Run `/atlassian-blueprint PROJ-123` again after reviewers leave feedback. The skill:

1. Reads comments from both Jira and Confluence
2. Proposes updates based on the feedback
3. Updates the plan in Confluence

### Phase 3 — Implement

Run `/atlassian-blueprint PROJ-123 implement` once the ticket is approved. The skill:

1. Verifies the ticket has been approved in Jira
2. Checks for any unaddressed feedback
3. Implements the plan (code changes, tests)
4. Creates a pull request and links it on the Jira ticket

## Requirements

- Claude Code CLI
- Atlassian MCP server (Jira + Confluence access)
- Git
- GitHub CLI (`gh`) — for creating pull requests

## License

MIT
