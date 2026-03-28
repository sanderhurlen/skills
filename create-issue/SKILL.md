---
name: create-issue
description: >
    Create issues or work items (GitHub or Azure DevOps) with proper context and description.
    Use when user wants to create a new issue, bug, task, or user story.
    Triggers on: "create issue", "new issue", "report bug", "new bug", "add to backlog".
argument-hint: '<description of the issue>'
allowed-tools: Bash, Read, Grep, Glob
---

# Create Issue / Work Item

Create well-structured issues with proper context and descriptions. Works with GitHub and Azure DevOps.

## Workflow

### Step 1: Detect the platform

```bash
git remote get-url origin
```

Parse the URL to determine the platform:

- **GitHub**: URL contains `github.com` → use `gh` CLI. Extract `owner/repo`.
- **Azure DevOps**: URL contains `dev.azure.com` → use `az repos` CLI. Extract `ORG_URL`, `PROJECT`, `REPO`.

### Step 2: Parse input

`$ARGUMENTS` is a natural language description of the issue/feature/bug. If empty, ask the user what they want to create and stop.

### Step 3: Determine issue type

Based on the description, suggest a type:

| Pattern                                             | GitHub Label / AzDO Type |
| --------------------------------------------------- | ------------------------ |
| Something is broken, wrong behavior, crash, error   | `bug` / **Bug**          |
| General work, implement, add, change, update        | `enhancement` / **Issue**|
| User-facing feature from user perspective           | `enhancement` / **User Story** |
| Small, well-scoped technical task                   | (no label) / **Task**    |

If ambiguous, ask the user to confirm the type before proceeding.

### Step 4: Check for duplicates

Search for existing open issues with similar titles:

#### GitHub:

```bash
gh issue list --state open --search "<key terms from title>" --limit 10
```

#### Azure DevOps:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.TeamProject] = '$PROJECT' AND [System.Title] CONTAINS '<key-terms>' AND [System.State] <> 'Closed' AND [System.State] <> 'Removed'" \
  --org "$ORG_URL" --project "$PROJECT" -o json
```

If potential duplicates are found, warn the user before proceeding.

### Step 5: Gather context (platform-specific)

#### Azure DevOps only — gather sprint and epic context (run in parallel):

**Resolve current sprint** (the `@CurrentIteration` macro does NOT work in the CLI — resolve explicitly):

```bash
az boards iteration team list --team "<Team>" --org "$ORG_URL" --project "$PROJECT" --timeframe current -o json --query "[0].{name:name, path:path}"
```

**Fetch available epics and features:**

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.WorkItemType] FROM WorkItems WHERE [System.TeamProject] = '$PROJECT' AND ([System.WorkItemType] = 'Epic' OR [System.WorkItemType] = 'Feature') AND [System.State] <> 'Closed' AND [System.State] <> 'Done' AND [System.State] <> 'Removed' ORDER BY [System.WorkItemType] ASC, [System.Title] ASC" \
  --org "$ORG_URL" --project "$PROJECT" -o json
```

Present epics/features as a numbered list and ask the user to pick one (or None). Suggest the best match based on the description but always confirm.

**Sprint assignment**: Default to current sprint. Ask user to confirm or pick a different one.

#### GitHub only — gather milestones and labels:

```bash
gh api repos/{owner}/{repo}/milestones --jq '.[].title'
gh label list
```

Ask the user if they want to assign a milestone or labels (optional).

### Step 6: Build the issue content

Based on the type, build structured content. Match the language used in the repo (check existing issues and commit messages).

**For Bugs:**

```markdown
## What happened
<what is happening>

## Expected behavior
<what should happen>

## Steps to reproduce
1. ...
2. ...

## Acceptance criteria
- [ ] The bug is fixed and verified
- [ ] <specific criterion>
```

If the user described a code-level bug, search the codebase for relevant files and mention them.

**For Enhancements / Issues / Tasks:**

```markdown
## What needs to be done
<description of the work and why>

## Acceptance criteria
- [ ] <concrete, testable criterion>
- [ ] <concrete, testable criterion>
```

**For User Stories:**

```markdown
## User story
As a [role], I want to [action] so that [benefit].

## Context
<background>

## Acceptance criteria
- [ ] <specific, testable scenario>
```

### Step 7: Present for approval

Show the user exactly what will be created:

```
## New <Type>
**Title**: <title>

**Description**:
<description>

**Labels**: <labels (GitHub) or Type/Sprint/Epic (Azure DevOps)>
```

Wait for explicit user approval. Accept edits. Do NOT create the issue until confirmed.

### Step 8: Create the issue

#### GitHub:

```bash
gh issue create \
  --title "<title>" \
  --body "$(cat <<'EOF'
<description>
EOF
)" \
  --label "<label>" \
  --milestone "<milestone-if-selected>"
```

#### Azure DevOps — create:

```bash
az boards work-item create \
  --type "<Type>" \
  --title "<Title>" \
  --description "<Description>" \
  --iteration "$PROJECT\\<Sprint Name>" \
  --org "$ORG_URL" \
  --project "$PROJECT" \
  -o json
```

Then, if acceptance criteria was provided:

```bash
az boards work-item update \
  --id <NEW_ID> \
  --fields "Microsoft.VSTS.Common.AcceptanceCriteria=<criteria>" \
  --org "$ORG_URL" --project "$PROJECT"
```

And if an epic/feature was selected:

```bash
az boards work-item relation add \
  --id <NEW_ID> \
  --relation-type "Parent" \
  --target-id <EPIC_ID> \
  --org "$ORG_URL"
```

### Step 9: Confirm creation

Show the created issue URL to the user.

## Rules

- **Auto-detect the platform** from `git remote get-url origin` — never hardcode org/project/repo
- **NEVER create the issue before getting explicit user approval** — present-then-confirm is mandatory
- Match the language used in the repo (existing issues, commit messages)
- Always check for duplicates before creating
- Acceptance criteria should be concrete and testable, not vague
- For bugs: search the codebase for relevant files if enough context is given
- Keep titles concise (under 80 chars)
- Do not invent sprint names, epic IDs, or work item data — always fetch from the API
- HTML is valid in Azure DevOps description fields — use `<br>` and `<ul><li>` if needed
