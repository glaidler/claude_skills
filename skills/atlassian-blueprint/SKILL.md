---
name: "atlassian-blueprint"
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


## Pre-Phase Check — Verify Azure Storage Config

Check `.claude/plans/.confluence-config.json` for `azure.storageAccount` and `azure.container`.

1. **If both keys exist** — verify access silently:
   ```
   az storage container show --account-name <storageAccount> --name <container> --auth-mode login
   ```
   - On failure: "I can't access the Azure storage account '<storageAccount>'. This might be a permissions issue — please check with your team."

2. **If either key is missing** — ask the user:
   - "What is the name of the Azure Storage account where I should upload UI previews?" → save as `azure.storageAccount`
   - "What is the blob container name?" → save as `azure.container`
   - Verify access as above before proceeding.

The config file now looks like:
```json
{
  "spaceKey": "TEAM",
  "pages": { ... },
  "azure": {
    "storageAccount": "myaccount",
    "container": "$web"
  },
  "snWebUxPath": "./en_ui"
}
```

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

**UI change identification is critical.** While exploring, explicitly catalogue every user-visible UI change the ticket will require. This list drives the mandatory mockup step later — if a UI change is missed here, it won't get mocked.

**E2E test discovery.** If the ticket involves UI changes, also explore for E2E test planning:

1. Check if `playwright.config.ts` exists somewhere — if not, the Playwright scaffold will need to be created as a prerequisite during implementation (see Phase 3, Step 5b).
2. List existing page objects in `e2e/pages/` that can be reused.
3. Trace which API endpoints the affected UI pages call by following the chain: page component → Redux actions/saga → `backend_helper.jsx` → `url_helper.jsx`. These endpoints determine which fixtures are needed.
4. Note which interactive elements (buttons, form inputs, links, data tables, modals) will need `data-testid` attributes for stable test selectors.

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

1. <Question or decision 1 that needs stakeholder input>
2. <Question or decision 2>
<If no decisions are needed, replace the numbered list with "No open questions — this one is straightforward.">

## UI Preview

<Include this section ONLY if the ticket involves user-visible UI changes. Delete it entirely for backend-only or non-visual work.>

[View all mockups](<azure-index-url>)

<List each mockup with its unique ID, deep link, and description:>
| ID | Preview | Description |
|----|---------|-------------|
| <JIRA_KEY>-M1 | [View](<mockup-1-url>) | <What this mockup shows> |
| <JIRA_KEY>-M2 | [View](<mockup-2-url>) | <What this mockup shows> |

## Risks & Impact

- <Plain-language risk 1, e.g. "Users will be logged out when we deploy this">
- <Plain-language risk 2, e.g. "This changes how notifications work, so we should let the support team know">
- <If low risk, say "Low risk — this change is isolated and doesn't affect other parts of the product.">

## Overall Feedback

> **Reviewers**: Use this section to leave any general comments about the plan — things that don't fit under a specific mockup or decision. For example: scope concerns, timeline thoughts, alternative approaches, or anything else you'd like to flag.
>
> *Highlight any text in this section and leave an inline comment with your overall feedback.*

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
- **E2E Tests** (include this section if the ticket has UI changes):
  - <User flow 1 to test — describe from the user's perspective>
  - <User flow 2>
  - **Fixtures needed:** <List API endpoints that need fixture JSON files in `en_ui/e2e/fixtures/`>
  - **Page objects:** <List new page objects to create in `en_ui/e2e/pages/` or existing ones to extend>
  - **`data-testid` attributes:** <List UI elements that need `data-testid` added during implementation>

## Technical Risks
- <Edge cases>
- <Potential breaking changes>
- <Dependencies or coordination needed>
```

Fill in every section thoroughly. The business summary sections should be understandable by anyone on the team. The technical sections should be detailed enough that a developer could implement the plan without additional context.

**Enforcement rule — UI mockup gate:** After writing the plan, review the "What Changes for Users" section. If ANY bullet point describes a visual or interface change (new buttons, layout changes, new screens, modified forms, style changes, etc.), then Step 4 (Build UI Mockups) is **mandatory** and every listed UI change must have a corresponding mockup. The plan MUST NOT be published to Confluence without mockups for all identified UI changes. There are no exceptions — if a mockup cannot be created, stop and ask the user for help rather than proceeding without it.

### Step 4 — Build UI Mockups

**This step is mandatory if the plan identifies ANY user-visible UI changes.** If the "What Changes for Users" section describes visual or interface changes, you MUST create mockups before proceeding. Do not skip this step. If you cannot create a mockup for a specific UI change (e.g., the relevant components don't exist in en_ui), stop and ask the user for guidance rather than skipping silently.

If the ticket has no UI-visible changes (backend-only, configuration, etc.), skip this step entirely.

#### 4a — Initialize the Mockups Project (first run only)

If `.claude/mockups/` does not already exist:

1. Scaffold a minimal React + Vite project:
   ```
   npm create vite@latest .claude/mockups -- --template react
   cd .claude/mockups && npm install
   ```
2. Install additional dependencies:
   ```
   cd .claude/mockups && npm install react-router-dom reactstrap bootstrap sass
   ```
3. Add `.claude/mockups/` to the project's `.gitignore` if not already present.

If `.claude/mockups/` already exists, just ensure dependencies are installed:
```
cd .claude/mockups && npm install
```

#### 4a.1 — Configure Vite and App.jsx for en_ui Theme

The mockups MUST use the same CSS theme as the production app. There are two files to configure:

**`App.jsx` — Theme import and routing:**
- Replace any `import "bootstrap/dist/css/bootstrap.min.css"` with the en_ui theme:
  ```javascript
  import "../../../en_ui/src/assets/scss/theme.scss";
  ```
  (Path is relative from `.claude/mockups/src/` → project root `en_ui/`.)
- Use `HashRouter` instead of `BrowserRouter`. Azure Blob Storage serves static files and does not support server-side routing, so SPA routes must use hash-based URLs (e.g., `index.html#/CS-5864-M1`).

**`vite.config.js` — Sass resolution:**

The en_ui `theme.scss` uses `@import "./node_modules/bootstrap/scss/..."` which resolves relative to the file's own directory. Vite needs a custom importer to redirect these paths to `en_ui/node_modules/`. The config also sets up `loadPaths` and a `~` alias so all SCSS import styles used in en_ui work correctly:

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'
import { fileURLToPath } from 'url'

const __dirname = path.dirname(fileURLToPath(import.meta.url))

// en_ui is at the project root, two levels up from .claude/mockups/
const enUiPath = path.resolve(__dirname, '..', '..', 'en_ui')

export default defineConfig({
  plugins: [react()],
  css: {
    preprocessorOptions: {
      scss: {
        api: 'modern-compiler',
        silenceDeprecations: ['mixed-decls', 'color-functions', 'global-builtin', 'import'],
        importers: [{
          findFileUrl(url) {
            // Redirect ./node_modules/... imports to en_ui/node_modules/...
            if (url.startsWith('./node_modules/')) {
              const resolved = path.join(enUiPath, url.slice(2))
              return new URL('file:///' + resolved.replace(/\\/g, '/'))
            }
            return null
          }
        }],
        loadPaths: [
          enUiPath,
          path.join(enUiPath, 'node_modules'),
          path.join(enUiPath, 'src'),
          path.join(enUiPath, 'src', 'assets', 'scss'),
        ],
      },
    },
  },
  resolve: {
    alias: {
      '~': path.join(enUiPath, 'node_modules'),
    },
  },
})
```

Both configurations are critical — without them, mockups will either fail to build or render without the correct branding.

#### 4b — Create Mockup Components

Each mockup gets a **unique ID** using the pattern `<JIRA_KEY>-M<n>` where `<n>` is a sequential number starting at 1. For example: `PROJ-123-M1`, `PROJ-123-M2`, `PROJ-123-M3`. These IDs are used everywhere — file names, URLs, the plan, and the Confluence feedback sections — so reviewers can reference a specific mockup unambiguously.

1. Create the directory `.claude/mockups/src/pages/<JIRA_KEY>/`.
2. For **each** UI change identified in the plan, assign the next available mockup ID and create a React component file named `<MOCKUP_ID>.jsx` (e.g., `PROJ-123-M1.jsx`). Each component should:
   - Import real components from `en_ui` (buttons, forms, layouts, etc.)
   - Show a **"Before"** and **"After"** view side by side, or just **"Proposed"** if it's a brand new screen
   - Use realistic sample data that matches the ticket's context
   - Include a clear header showing the mockup ID and a plain-language description of what changed
3. Create an index page at `.claude/mockups/src/pages/<JIRA_KEY>/index.jsx` that:
   - Displays the Jira ticket title and key at the top
   - Lists all mockups with their IDs, descriptions, and direct links
4. Update `.claude/mockups/src/App.jsx` to add routes using **HashRouter**:
   - `/` → index page for this ticket
   - `/<MOCKUP_ID>` → individual mockup page (one route per mockup)

   The URL structure becomes `index.html#/<MOCKUP_ID>` which works on static hosting without server-side routing.

#### 4c — Build the Mockup

**IMPORTANT — Base path:** The `--base` flag must match the full URL path in Azure Blob Storage, which is `/<container>/<JIRA_KEY>/` (e.g., `/mockups/CS-5864/`), NOT just `/<JIRA_KEY>/`. If the base path is wrong, assets (JS/CSS) will fail to load and the page will be blank.

**IMPORTANT — Windows Git Bash:** On Windows with Git Bash, paths starting with `/` get auto-converted to Windows paths (e.g., `/mockups/` becomes `C:/Program Files/Git/mockups/`). Prefix the command with `MSYS_NO_PATHCONV=1` to prevent this:

```
cd .claude/mockups && MSYS_NO_PATHCONV=1 npm run build -- --base=/<container>/<JIRA_KEY>/
```

On failure: "I ran into a problem building the UI previews. Here's what went wrong: [friendly one-sentence summary of the build error]." Then stop and ask the user how to proceed.

#### 4d — Upload to Azure Blob Storage

Read `azure.storageAccount` and `azure.container` from `.claude/plans/.confluence-config.json`, then upload:

```
az storage blob upload-batch \
  --account-name <storageAccount> \
  --destination <container>/<JIRA_KEY> \
  --source .claude/mockups/dist \
  --overwrite \
  --auth-mode key
```

**Note:** Use `--auth-mode key` (not `login`). The `login` mode requires RBAC role assignments (Storage Blob Data Contributor) which may not be configured. The `key` mode uses the storage account key and works immediately.

On failure: "I couldn't upload the UI previews. This might be a permissions issue — please check with your team."

#### 4e — Record the Preview URLs

Since the mockups use **HashRouter** for client-side routing, all routes are served from `index.html` with hash fragments. Construct the URLs using the **blob endpoint** (not the static website endpoint):

- **Index URL:** `https://<storageAccount>.blob.core.windows.net/<container>/<JIRA_KEY>/index.html`
- **Per-mockup URL:** `https://<storageAccount>.blob.core.windows.net/<container>/<JIRA_KEY>/index.html#/<MOCKUP_ID>`

**Do NOT use the static website endpoint** (`*.z13.web.core.windows.net`) unless the container is `$web`. For named containers like `mockups`, use the blob endpoint directly.

Store the index URL and the list of mockup IDs with their descriptions and individual URLs. These will be used in the plan file, the Confluence page feedback sections, and the Jira comment.

### Step 5 — Save the Plan Locally

Save the plan file to `.claude/plans/<JIRA_KEY>.md`. No git operations are needed at this stage — the plan is saved locally and will be published to Confluence in the next step.

### Step 6 — Publish Plan to Confluence

1. **Determine the Confluence space**:
   - Check if a space preference has been stored previously in `.claude/plans/.confluence-config.json`
   - If no preference exists, call `getConfluenceSpaces` to list available spaces
   - Ask the user: "Where should I publish the plan for review? Here are the available spaces:" and list the space names
   - Store the user's choice in `.claude/plans/.confluence-config.json` as `{"spaceKey": "<KEY>"}` for future runs

2. **Find or create the "Plans" folder page**:
   - Check if a `plansFolderId` is already stored in `.claude/plans/.confluence-config.json`
   - If not, search for an existing page titled **"Plans"** in the selected space using `searchConfluenceUsingCql` with CQL: `type = page AND space = "<spaceKey>" AND title = "Plans"`
   - If found, store its page ID as `plansFolderId` in `.claude/plans/.confluence-config.json`
   - If not found, create it by calling `createConfluencePage` with title "Plans", an empty body (or a brief description like "This folder contains implementation plans for Jira tickets."), and no `parentId`. Store the new page's ID as `plansFolderId` in `.claude/plans/.confluence-config.json`

3. **Create two Confluence pages** (business summary + technical detail):

   > **Important:** All Atlassian MCP API calls during plan publishing must be made **sequentially** (one at a time). Do not make parallel calls to the Atlassian MCP — this can trigger rate limiting from Cloudflare.

   **Step 3a — Create the business summary page:**
   - Call `createConfluencePage` with:
     - Space key from config
     - `parentId` set to the `plansFolderId` from config
     - Title: `Plan: <JIRA_KEY> - <Jira Title>`
     - Body: only the **business sections** of the plan — everything **above** the `---` technical divider. This includes: Summary, What Changes for Users, Decisions Needed, UI Preview (if applicable), Risks & Impact.
     - If the ticket has UI mockups, append a per-mockup feedback section after the UI Preview table:

       > ## Mockup Feedback
       >
       > Please leave feedback on each mockup below. Reference the mockup ID so we know exactly which one you mean.
       >
       > ---
       >
       > ### <JIRA_KEY>-M1 — <short description>
       > [View mockup](<mockup-1-url>)
       >
       > *Leave a comment on this section with your feedback for this mockup.*
       >
       > ---
       >
       > ### <JIRA_KEY>-M2 — <short description>
       > [View mockup](<mockup-2-url>)
       >
       > *Leave a comment on this section with your feedback for this mockup.*
       >
       > ---
       >
       > <Repeat for each mockup>

     Each mockup gets its own headed section so reviewers can leave **inline Confluence comments** directly on the relevant heading. This makes it clear which mockup the feedback applies to.

     Omit the "Mockup Feedback" section entirely for non-UI tickets.

   - **Always include** an "Overall Feedback" section at the end of the business summary. This section is present on every plan — not just UI tickets — so reviewers have a clear place for general comments about scope, approach, timeline, or anything else:

       > ## Overall Feedback
       >
       > Use this section to leave any general comments about the plan — things that don't fit under a specific mockup or decision. For example: scope concerns, timeline thoughts, alternative approaches, or anything else you'd like to flag.
       >
       > *Highlight any text in this section and leave an inline comment with your overall feedback.*

   - At the very bottom of the business page, add a link to the technical detail page (use a placeholder initially; update it after the technical page is created):

       > ---
       >
       > **Technical detail:** [Plan: <JIRA_KEY> - Technical Detail](<technical-page-url>)

   **Step 3b — Create the technical detail page:**
   - Call `createConfluencePage` with:
     - Space key from config
     - `parentId` set to the **business summary page ID** (from Step 3a) — making it a child page
     - Title: `Plan: <JIRA_KEY> - Technical Detail`
     - Body: everything **below** the `---` technical divider from the plan file. This includes: Jira Details, Analysis, Approach, Implementation Steps, Files to Modify, New Files, Testing Strategy, Technical Risks.
     - At the very top of the technical page, add a back-link to the business summary page:

       > Back to business summary: [Plan: <JIRA_KEY> - <Jira Title>](<business-page-url>)
       >
       > ---

   **Step 3c — Update the business page link:**
   - After the technical page is created and you have its URL, call `updateConfluencePage` on the business summary page to replace the placeholder technical detail link with the real URL.

   **Step 3d — Store both page IDs** in `.claude/plans/.confluence-config.json`:
   ```json
   {
     "pages": {
       "<JIRA_KEY>": {
         "pageId": "<business-summary-page-id>",
         "technicalPageId": "<technical-detail-page-id>",
         "syncState": "full",
         "localPlanHash": "<SHA-256 of the local .md file>",
         "lastSyncedAt": "<ISO 8601 timestamp>"
       }
     }
   }
   ```

3. **Create Jira remote link to the Confluence page** (automatic bidirectional linking):
   - Look for Jira credentials in this priority order:
     1. **Repo config** (shared, preferred): read `jira.email`, `jira.apiToken`, `jira.baseUrl` from `.claude/plans/.confluence-config.json`
     2. **User-level fallback**: read from `~/.claude/projects/<sanitized-project-dir>/jira-credentials.json` (where the sanitized dir replaces path separators with dashes)
   - If credentials are found from either source, run:
     ```bash
     curl -s -X POST \
       "<baseUrl>/rest/api/3/issue/<JIRA_KEY>/remotelink" \
       -H "Authorization: Basic $(echo -n '<email>:<apiToken>' | base64 -w 0)" \
       -H "Content-Type: application/json" \
       -d '{
         "object": {
           "url": "<confluence-page-url>",
           "title": "<confluence-page-title>",
           "icon": {
             "url16x16": "<baseUrl>/wiki/favicon.ico",
             "title": "Confluence"
           }
         },
         "relationship": "Confluence Page"
       }'
     ```
   - On success (HTTP 200/201): proceed silently — the link now appears in the Jira issue under "Confluence pages".
   - On failure or if credentials file doesn't exist: skip silently — the user can link manually later.

4. **Post the plan summary and questions on the Jira ticket**:
   - Call `addCommentToJiraIssue` to post a comment that includes the business-facing sections from the plan. Use this format:

     > ## Plan ready for review
     >
     > **Summary:** <Summary section from the plan>
     >
     > **What changes for users:**
     > - <bullet points from the "What Changes for Users" section>
     >
     > **UI Preview:** [View mockups](<azure-preview-url>)
     > <Include this line ONLY if UI mockups were built in Step 4. Omit entirely for non-UI tickets.>
     >
     > **Risks & impact:**
     > - <bullet points from the "Risks & Impact" section>
     >
     > ### Decisions needed
     > 1. <question 1>
     > 2. <question 2>
     >
     > Full plan details: [Plan: <JIRA_KEY> - <Jira Title>](<confluence-page-url>)

   - This lets stakeholders review and respond directly in Jira without needing to visit Confluence. The Confluence link is still included for anyone who wants the full technical detail.

### Step 7 — Transition the Jira Ticket

1. Call `getTransitionsForJiraIssue` to get available transitions
2. Look for a transition that moves the ticket to a review-like status (e.g., "In Review", "Planning", "In Progress")
3. If a suitable transition is found, call `transitionJiraIssue` to move the ticket
4. If no obvious transition exists, tell the user: "I've published the plan but I'm not sure which status to move the ticket to. Could you update the ticket status in Jira to indicate it's ready for review?"

### Step 8 — Tell the User What Happened

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

1. Read the Confluence config from `.claude/plans/.confluence-config.json` to get the page ID(s) for this Jira issue.
   - The config entry may be in one of three formats:
     - **String** (legacy): `"CS-XXXX": "12345"` — a single page ID containing the entire plan.
     - **Object without technicalPageId** (legacy): `"CS-XXXX": { "pageId": "12345", ... }` — a single page containing the entire plan.
     - **Object with technicalPageId** (current): `"CS-XXXX": { "pageId": "12345", "technicalPageId": "12346", ... }` — two pages: business summary + technical detail.
   - Extract `pageId` (the business summary page) and `technicalPageId` (the technical detail page, if present) from whichever format is found.
2. If no config exists, search Confluence for the plan page:
   - Call `getConfluenceSpaces` then search for a page titled `Plan: <JIRA_KEY>*`
3. If no plan page is found, tell the user: "I couldn't find an existing plan for this ticket. Would you like me to create one? Run `/engage-blueprint <JIRA_KEY>` on a ticket in 'To Do' status to start fresh."

### Step 2 — Read Feedback

Gather feedback from Jira and Confluence (both footer and inline comments). For two-page plans, check **both** the business summary page and the technical detail page for comments.

> **Important:** All Atlassian MCP API calls during feedback gathering must be made **sequentially** (one at a time). Do not make parallel calls to the Atlassian MCP — this can trigger rate limiting from Cloudflare.

1. **Jira comments** — Call `getJiraIssue` with the cloud ID and issue key, and read any comments on the ticket. Look for replies to the "Plan ready for review" comment that contain stakeholder answers to the decisions needed.
2. **Confluence footer comments** — Call `getConfluencePageFooterComments` with the business summary page ID to get reviewer comments left on the plan page. If `technicalPageId` exists, also call `getConfluencePageFooterComments` on the technical page.
3. **Confluence inline comments** — Call `getConfluencePageInlineComments` with the business summary page ID and `resolutionStatus: "open"` to get unresolved inline comments left directly on plan content (mockup headings, the "Overall Feedback" section, or any other text). If `technicalPageId` exists, also call `getConfluencePageInlineComments` on the technical page to get inline comments on the technical sections. For each inline comment, also call `getConfluenceCommentChildren` with `commentType: "inline"` to check for existing replies — if a reply contains "Addressed in plan update", treat the comment as already handled and skip it.
4. Call `getConfluencePage` on the business summary page to read the current plan content. If `technicalPageId` exists, also call `getConfluencePage` on the technical page.

Combine feedback from all sources. If the same question was answered in multiple places, prefer the most recent response.

**Categorize feedback by section:** When processing Confluence comments, identify which section each comment is attached to. For inline comments, use the `textSelection` property (the highlighted text) to determine which section the comment targets — match it against mockup headings (`### <JIRA_KEY>-M1 — ...`) or the "Overall Feedback" heading. For footer comments, infer the section from the comment content.

- **Mockup feedback** — inline or footer comments on "Mockup Feedback" headings (e.g., `### PROJ-123-M1 — ...`). Tag with the mockup ID for routing to the correct component during rebuild. Present grouped by ID:
  - **PROJ-123-M1** (Login form redesign): "The submit button should be more prominent" — Jane [inline]
  - **PROJ-123-M2** (Dashboard layout): "Can we add a sidebar?" — Tom [inline]

- **Overall feedback** — inline or footer comments on the "Overall Feedback" heading. These are general plan comments (scope, approach, timeline, alternatives). Present separately:
  - **Overall**: "Can we phase this — do the form changes first and the notification changes in a follow-up?" — Sarah [inline]
  - **Overall**: "This looks good but we should check with the mobile team first" — Tom [footer]

- **Other comments** — comments on any other section. Present with the section name for context.

### Step 3 — Update the Plan

1. Read the current plan from `.claude/plans/<JIRA_KEY>.md` (or fetch from Confluence if the local file is missing)
2. Analyze the feedback:
   - Summarize each piece of feedback for the user in plain language
   - Suggest changes to address each comment
3. Ask the user if the proposed changes look good
4. Update `.claude/plans/<JIRA_KEY>.md` with the changes

### Step 4 — Rebuild UI Mockups (if UI sections changed)

If the plan update in Step 3 affected any UI-related sections ("What Changes for Users", "UI Preview", or any mockup-relevant content), or if mockup-specific feedback was received:

1. Update only the affected mockup components in `.claude/mockups/src/pages/<JIRA_KEY>/`, targeting each by its mockup ID (e.g., update `PROJ-123-M1.jsx` for feedback tagged to `PROJ-123-M1`). Leave unchanged mockups untouched.
2. Rebuild (use `MSYS_NO_PATHCONV=1` on Windows Git Bash):
   ```
   cd .claude/mockups && MSYS_NO_PATHCONV=1 npm run build -- --base=/<container>/<JIRA_KEY>/
   ```
3. Re-upload to Azure (same URL, overwrite):
   ```
   az storage blob upload-batch \
     --account-name <storageAccount> \
     --destination <container>/<JIRA_KEY> \
     --source .claude/mockups/dist \
     --overwrite \
     --auth-mode key
   ```
4. Tell the user: "I've updated the UI previews to reflect the changes. You can see them at the same link."

If no UI-related sections changed, skip this step.

### Step 5 — Save and Upload Updates

Save the updated plan to `.claude/plans/<JIRA_KEY>.md` locally, then update Confluence. For two-page plans, split the updated content and update each page separately. Because inline comments are anchored to specific text via `textSelection`, updating the page content can cause inline comments to become **dangling** (losing their anchor) if the text they were attached to changed. To prevent feedback loss, follow this sequence:

> **Important:** All Atlassian MCP API calls during plan updates must be made **sequentially** (one at a time). Do not make parallel calls to the Atlassian MCP — this can trigger rate limiting from Cloudflare.

#### 5a — Snapshot inline comments before update

Before calling `updateConfluencePage`, read all open inline comments from **both** the business summary page and the technical detail page (if it exists) using `getConfluencePageInlineComments` with `resolutionStatus: "open"`. For each comment, record:
- Comment ID
- Which page it came from (business or technical)
- Author
- Body text
- `textSelection` (the highlighted text it was anchored to)
- Whether it was addressed in this iteration (from Step 2 categorization)

#### 5b — Embed unaddressed feedback into the updated page

For any **unaddressed** inline comments (feedback that was not incorporated in this update — e.g., deferred decisions, questions still pending), embed them directly into the relevant section of the updated plan content as attributed blockquotes. This ensures they survive the page update even if the original anchor text changes.

Format each embedded comment as:

> **Reviewer feedback** (<author>, <date>): "<original comment text>"

Place each embedded comment in the section it was originally attached to:
- Comments on mockup headings → under the relevant `### <JIRA_KEY>-M<n>` heading
- Comments on "Overall Feedback" → in the "Overall Feedback" section
- Comments on "Decisions Needed" → in the "Decisions Needed" section
- Comments on technical sections (from the technical page) → in the matching section below the `---` divider
- Comments on other sections → in the matching section

If the target section was removed or heavily rewritten, place the comment in an "Unresolved Feedback" section appended just above the technical divider:

> ## Unresolved Feedback
>
> The following reviewer comments could not be anchored to their original sections after this update:
>
> - "<comment text>" — <author> (originally on: <textSelection snippet>)

#### 5c — Update the Confluence pages

Split the updated plan content at the `---` technical divider:

1. **Update the business summary page** — Call `updateConfluencePage` with the business page ID and the business sections (everything above the `---` divider), including the link to the technical detail page at the bottom.

2. **Update the technical detail page** — If `technicalPageId` exists, call `updateConfluencePage` with the technical page ID and the technical sections (everything below the `---` divider), including the back-link to the business page at the top.

3. **Migrate legacy single-page plans** — If the config entry has no `technicalPageId` (legacy single-page format), create a new technical detail child page using the same approach as Phase 1 Step 3b, then update the config to add the `technicalPageId`.

4. **Update sync state** — After both pages are updated, compute the SHA-256 hash of the local `.md` file and update `localPlanHash` and `lastSyncedAt` in `.confluence-config.json`.

#### 5d — Acknowledge addressed feedback

After updating the pages, reply to each **addressed** inline and footer comment (on whichever page it was found):

- **Inline comments**: For each addressed inline comment, call `createConfluenceInlineComment` with `parentCommentId` set to the original comment's ID. Use this format: "Addressed in plan update — [one-sentence summary of the change made]." For example: "Addressed in plan update — made the submit button larger and changed colour to primary blue."
- **Footer comments**: Optionally reply to footer comments using `createConfluenceFooterComment` with `parentCommentId` to acknowledge feedback was addressed.

This creates an audit trail: reviewers can see their original comment plus the reply confirming it was handled.

### Step 6 — Tell the User

Tell the user:
- "I've updated your plan based on the review feedback."
- Summarize the changes in bullet points
- If mockups were rebuilt: "I've also updated the UI previews — same link as before."
- Provide the Confluence page link: "You can see the updated plan here: [link]"
- "Once the reviewer is happy and moves the ticket to an approved status, run `/engage-blueprint <JIRA_KEY> implement` to start building."

---

## Phase 3: Implement

When the Jira ticket is in an approved status and `implement` was explicitly passed. This phase writes real code.

### Step 1 — Verify the Ticket is Approved

1. Fetch the Jira issue using `getJiraIssue` and check its current status
2. Check whether the status indicates approval:
   - If status is **"Approved"**, **"Plan Approved"**, **"Ready for Dev"**, **"Selected for Development"**, or similar → proceed to Step 2
   - If status is **anything else** → **STOP**. Do NOT offer to proceed. Tell the user: "This ticket hasn't been approved yet. Before I can start building, someone needs to review the plan and move the ticket to an approved status in Jira. You can find the plan here: [Confluence page link]. Once the ticket is approved, come back and run this command again."

This is a hard gate. Under no circumstances should implementation proceed without the ticket being in an approved status.

**Escape valve**: If the user says they are the only person on the project and cannot get someone else to approve, tell them: "You can move the ticket to the approved status yourself in Jira, and then run this command again."

### Step 2 — Check for Unaddressed Feedback

Before building, check whether any comments were left after the plan was last updated. For two-page plans, check **both** pages.

> **Important:** All Atlassian MCP API calls during feedback checking must be made **sequentially** (one at a time). Do not make parallel calls to the Atlassian MCP.

1. Read the Confluence business summary page using `getConfluencePage` — note the page's `version.when` (last modified timestamp). If `technicalPageId` exists, also read the technical detail page.
2. Read Confluence footer comments using `getConfluencePageFooterComments` on the business summary page. If `technicalPageId` exists, also check the technical page.
3. Read Confluence inline comments using `getConfluencePageInlineComments` with `resolutionStatus: "open"` on the business summary page. If `technicalPageId` exists, also check the technical page. For each open inline comment, call `getConfluenceCommentChildren` with `commentType: "inline"` to check for replies. An inline comment is **addressed** if it has a reply containing "Addressed in plan update"; otherwise it is **unaddressed**.
4. Read Jira comments from the ticket using `getJiraIssue`.
5. Compare timestamps: if any footer or Jira comments from either source are **newer** than the plan's last modified timestamp, they may not have been incorporated. For inline comments, use the reply-based check from step 3 instead of timestamps.

If unaddressed comments are found (from any source):
- List them for the user in plain language: "I found feedback that came in after the plan was last updated:"
  - 1. "<comment summary>" — <author> [inline/footer/Jira]
  - 2. "<comment summary>" — <author> [inline/footer/Jira]
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

### Step 5b — Generate E2E Tests

If the plan's Testing Strategy includes an **E2E Tests** section, generate Playwright end-to-end tests for the UI changes. Skip this step entirely if there are no E2E tests in the plan.

Tell the user: "I'm now setting up automated browser tests for the changes."

#### 1. Bootstrap Playwright (one-time)

If `en_ui/e2e/playwright.config.ts` does **not** exist, the Playwright scaffold needs to be created first:

```bash
cd en_ui && npm install && npx playwright install chromium
```

Then create the scaffold structure. The following files must exist (create from the established templates in `en_ui/e2e/`):
- `e2e/playwright.config.ts` — Two projects: `mock` (CI, uses `USE_MOCK_API=true`) and `live` (post-deployment, hits real backend)
- `e2e/utils/api-mock.ts` — `setupApiMocks(page, routeMap)` utility that registers `page.route()` handlers from JSON fixture files. In mock mode, unmatched API calls return HTTP 500 with a descriptive error. In live mode, this is a no-op.
- `e2e/utils/auth.ts` — `setupAuth(page)` utility that injects localStorage values (`authUser`, `selectedSchoolId`, `userRole`, `InternalUserId`, `actingAsRole`) to bypass login for non-login tests. Also exports `loginViaApi(page)` for live mode.
- `e2e/fixtures/auth/login-success.json`, `e2e/fixtures/auth/user-profile.json`, `e2e/fixtures/common/menu-options.json` — Seed fixtures for auth flows.
- `e2e/pages/login.page.ts` — Login page object.
- `e2e/tests/auth/login.spec.ts` — Seed login tests.

If the scaffold already exists, just ensure Playwright is installed: `cd en_ui && npx playwright install chromium`.

#### 2. Add `data-testid` attributes

During Step 5 (when implementing the feature code), `data-testid` attributes should have been added to the elements listed in the plan's E2E test section. Verify they are in place. If any are missing, add them now.

Selector priority for page objects: `data-testid` > `role` > `text/placeholder` > never CSS classes (Bootstrap classes change).

#### 3. Create/update fixtures

For each API endpoint listed in the plan's "Fixtures needed" section:

1. Check if a fixture already exists in `e2e/fixtures/`
2. If not, create one:
   - Derive the response shape by tracing: page component → saga `yield call(backendHelperFn)` → `backend_helper.jsx` function → axios response shape
   - Use realistic sample data that matches the test scenario
   - Save as plain JSON in `e2e/fixtures/<domain>/<endpoint-name>.json`

Each fixture file contains exactly what the API returns (the response body). Keep fixture data minimal but realistic.

#### 4. Create/update page objects

For each page or component under test:

1. Check if a page object exists in `e2e/pages/`
2. If not, create one following this pattern:
   - Export a class named `<PageName>Page`
   - Constructor takes a `Page` instance and sets up locators
   - Use stable selectors: `data-testid` → `getByRole` → `getByPlaceholder`/`getByText` → never CSS classes
   - Include navigation methods (`goto()`) and action methods (`fillForm()`, `submit()`)
   - Include locator properties for key elements that tests will assert against

#### 5. Write test specs

Create test files in `en_ui/e2e/tests/<domain>/`:

- Import the page object and `setupApiMocks` from `../../utils/api-mock`
- Import `setupAuth` from `../../utils/auth` for tests that need pre-authenticated state
- Call `setupApiMocks(page, routeMap)` with the fixture map for that test's API dependencies
- For non-login tests, call `setupAuth(page)` before navigating
- Test the happy path first, then error/edge cases
- Use descriptive `test.describe` and `test()` names that read like user stories

#### 6. Run E2E tests

```bash
cd en_ui && USE_MOCK_API=true npx playwright test --config=e2e/playwright.config.ts --project=mock
```

All tests must pass before proceeding. If tests fail, fix the issues (missing fixtures, wrong selectors, timing issues) and re-run.

Save the E2E test files as a separate commit:
```
git add en_ui/e2e/
git commit -m "Add E2E tests for <JIRA_KEY>"
```

### Step 6 — Final Review

After implementation is complete:

1. Run the full test suite if one exists, including E2E tests if they were generated in Step 5b. Tell the user: "I'm running tests to make sure everything works."
   - For E2E tests: `cd en_ui && USE_MOCK_API=true npx playwright test --config=e2e/playwright.config.ts --project=mock`
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
  "plansFolderId": "plans-parent-page-id",
  "pages": {
    "PROJ-123": "confluence-page-id-here",
    "PROJ-456": {
      "pageId": "business-summary-page-id",
      "technicalPageId": "technical-detail-page-id",
      "syncState": "full",
      "localPlanHash": "<SHA-256 of the local .md file>",
      "lastSyncedAt": "<ISO 8601 timestamp>"
    }
  },
  "azure": {
    "storageAccount": "myaccount",
    "container": "$web"
  },
  "snWebUxPath": "./en_ui",
  "jira": {
    "email": "team@example.com",
    "apiToken": "ATATT3x...",
    "baseUrl": "https://yoursite.atlassian.net"
  }
}
```

**Page entry formats** (all three must be supported for backward compatibility):
- **String** (legacy): `"PROJ-123": "12345"` — single page ID
- **Object without technicalPageId** (legacy): `"PROJ-123": { "pageId": "12345", "syncState": "full", ... }` — single page with sync tracking
- **Object with technicalPageId** (current): as shown above — two-page format with business summary + technical detail

To extract the business page ID from any format, use: if the value is a string, use it directly as the page ID; if it's an object, use the `pageId` field.

> **Security note:** The `jira` block stores a shared team API token in the repo. This is intentional for team-wide automation. Treat it as a shared credential — rotate it if team membership changes.

This file is created automatically on first run when the user selects a Confluence space. Azure storage and en_ui settings are added when first needed for UI mockups.

---

## Confluence Upload Resilience

The Atlassian MCP proxy routes through Cloudflare, which enforces a Web Application Firewall (WAF). Large page bodies — especially those containing security-related technical terminology — can trigger a Cloudflare block. The **two-page architecture** (business summary + technical detail as a child page) is the primary defence against this, since each page body is significantly smaller than a combined single-page upload.

### Sequential API Calls

**All Atlassian MCP API calls must be made sequentially** — one at a time, never in parallel. Concurrent requests to the Atlassian MCP can trigger Cloudflare rate limiting, which is a separate issue from WAF content blocking. This applies to all phases: creating pages, reading comments, updating content, and posting replies.

### Upload Strategy

When publishing or updating Confluence pages, use this approach:

1. **Always use the two-page architecture** — Business summary content goes to the main page; technical detail goes to the child page. This keeps each upload well under the WAF size threshold.

2. **On Cloudflare block** (HTTP block / "Attention Required" response) for the technical detail page — **Do not retry the same payload immediately.** Instead:
   - Wait at least 2 minutes before retrying (Cloudflare rate limits typically reset after 1–2 minutes).
   - On retry, if the page body is very large, attempt to split it into two child pages:
     - `Plan: <JIRA_KEY> - Technical Detail (Part 1)` — Analysis, Approach, Implementation Steps
     - `Plan: <JIRA_KEY> - Technical Detail (Part 2)` — Files to Modify, New Files, Testing Strategy, Technical Risks
   - If the split upload also fails, record `syncState: "partial"` and tell the user: "Most of the plan uploaded successfully, but the full technical detail was too large for Confluence. The complete plan is in the local file at `.claude/plans/<JIRA_KEY>.md`."

3. **Record the sync state** — After any upload, record the outcome in `.claude/plans/.confluence-config.json`:
   ```json
   {
     "pages": {
       "PROJ-123": {
         "pageId": "12345",
         "technicalPageId": "12346",
         "syncState": "full",
         "localPlanHash": "<SHA-256 of the local .md file contents>",
         "lastSyncedAt": "<ISO 8601 timestamp>"
       }
     }
   }
   ```
   - `syncState: "full"` — Both Confluence pages have the complete plan.
   - `syncState: "partial"` — Business summary uploaded successfully; technical detail may be incomplete. The local file is the source of truth for technical detail.

### Plan Parity Check

At the **start of Phase 2 (Review & Adjust)** and **Phase 3 (Implement)**, before doing any other work, check whether the Confluence pages are in sync with the local plan:

1. **Read the local plan** from `.claude/plans/<JIRA_KEY>.md` and compute its SHA-256 hash.
2. **Compare** against `localPlanHash` in the config:
   - If the hashes **match** and `syncState` is `"full"` → Confluence is up to date. Proceed normally.
   - If the hashes **don't match** → The local plan has been edited since the last sync. Re-upload both pages to Confluence (split at the `---` divider). Update `localPlanHash` and `lastSyncedAt` on success.
   - If `syncState` is `"partial"` → Confluence has incomplete technical detail from a previous failed upload. Attempt a re-upload of the technical detail page. If it succeeds, update `syncState` to `"full"`. If it fails again, proceed with the local plan as source of truth.

3. **If the local file is missing** but a Confluence page exists:
   - Fetch the content from both the business summary page and the technical detail page (if it exists) from Confluence.
   - Reconstruct the full plan (business sections + `---` divider + technical sections) and save it to `.claude/plans/<JIRA_KEY>.md`.
   - If `syncState` was `"partial"`, warn the user: "I recovered the plan from Confluence, but the technical detail may be incomplete. Please check the plan before proceeding."

### Implementation Phase Source of Truth

During **Phase 3 (Implement)**, **always read the plan from the local file** (`.claude/plans/<JIRA_KEY>.md`), never from Confluence. The local file is guaranteed to have the full technical detail regardless of `syncState`. Only fall back to Confluence if the local file is missing.
