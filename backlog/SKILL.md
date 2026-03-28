---
name: backlog
description: >
    Query and display backlog items or open issues (GitHub or Azure DevOps).
    Use when user wants to see the backlog, check what's in the sprint, find work items, or look for tasks.
    Triggers on: "backlog", "sprint items", "what's in the sprint", "show tasks", "my items", "open issues".
argument-hint: '[mine|bugs|unassigned|sprint|<search-term>]'
allowed-tools: Bash
---

# Backlog / Issue List

Query and display open issues or backlog items. Works with GitHub and Azure DevOps.

## Workflow

### Step 1: Detect the platform

```bash
git remote get-url origin
```

Parse the URL to determine the platform:

- **GitHub**: URL contains `github.com` → use `gh` CLI. Extract `owner/repo`.
- **Azure DevOps**: URL contains `dev.azure.com` → use `az repos` CLI. Extract `ORG_URL`, `PROJECT`, `REPO`.

### Step 2: Parse arguments

Interpret `$ARGUMENTS` to determine the query filter:

| Input                      | Filter                               |
| -------------------------- | ------------------------------------ |
| _(empty)_                  | Current sprint / all open issues     |
| `mine`, `my`               | Assigned to me                       |
| `sprint`, `current`        | Current sprint (AzDO) / open (GitHub)|
| `bugs`, `bug`              | Bugs only                            |
| `unassigned`               | Unassigned items                     |
| Anything else              | Search by title/keyword              |

### Step 3: Fetch items

#### GitHub:

```bash
# All open issues (default)
gh issue list --state open --limit 30

# Mine
gh issue list --state open --assignee @me --limit 30

# Bugs
gh issue list --state open --label bug --limit 30

# Unassigned
gh issue list --state open --assignee "" --limit 30

# Search
gh issue list --state open --search "<term>" --limit 30
```

#### Azure DevOps:

**First, resolve the current sprint** (`@CurrentIteration` does NOT work in the CLI):

```bash
az boards iteration team list --team "<Team>" --org "$ORG_URL" --project "$PROJECT" --timeframe current -o json --query "[0].{name:name, path:path}"
```

Then query (use the resolved iteration path, e.g. `Project\\Sprint 29`):

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo], [System.WorkItemType], [Microsoft.VSTS.Common.Priority], [System.Tags] FROM WorkItems WHERE [System.TeamProject] = '$PROJECT' AND [System.IterationPath] = '$PROJECT\\<Sprint Name>' AND [System.State] <> 'Closed' AND [System.State] <> 'Removed' ORDER BY [Microsoft.VSTS.Common.Priority] ASC, [System.State] ASC" \
  --org "$ORG_URL" --project "$PROJECT" -o json
```

Apply additional WHERE clauses as needed:

- **mine**: `AND [System.AssignedTo] = @Me`
- **bugs**: `AND [System.WorkItemType] = 'Bug'`
- **unassigned**: `AND [System.AssignedTo] = ''`
- **search**: Remove iteration filter, add `AND [System.Title] CONTAINS '<term>'`

For Azure DevOps, after getting IDs, fetch details for each:

```bash
az boards work-item show --id <ID> --org "$ORG_URL" --project "$PROJECT" \
  --query "{id:id, title:fields.\"System.Title\", state:fields.\"System.State\", assignedTo:fields.\"System.AssignedTo\".displayName, type:fields.\"System.WorkItemType\", priority:fields.\"Microsoft.VSTS.Common.Priority\", tags:fields.\"System.Tags\"}" -o json
```

If there are more than 20 items, only fetch details for the first 20 and note the total count.

### Step 4: Display formatted

Present as a clean markdown table:

#### GitHub:

```
## Open Issues

| # | Title | Assignee | Labels | Updated |
|---|-------|----------|--------|---------|
| #42 | Fix null pointer in loader | @alice | bug | 2d ago |
| #38 | Add dark mode | -- | enhancement | 5d ago |
```

#### Azure DevOps:

```
## Backlog — Current Sprint

| # | Type | Title | Assigned | State | Pri |
|---|------|-------|----------|-------|-----|
| 7142 | 🐛 Bug | Null pointer in loader | Jakob | Active | 1 |
| 7089 | 📋 Task | Add i18n to settings | -- | New | 2 |
```

Work item type emoji (Azure DevOps):

| Type       | Emoji |
| ---------- | ----- |
| Bug        | 🐛    |
| Task       | 📋    |
| User Story | 📖    |
| Feature    | 🚀    |
| Epic       | ⚡    |

Formatting rules:

- Sort by priority (AzDO) or last updated (GitHub)
- Show `--` for unassigned items
- Truncate titles to ~60 chars if needed
- If tags exist (AzDO), show in parentheses after the title

### Step 5: Offer follow-up actions

After displaying the table, suggest:

- "Want to see details on any of these? Give me the issue number."
- "I can also filter: `mine`, `bugs`, `unassigned`, or a search term."

If the user gives an issue/item number as follow-up, fetch its full details (description, acceptance criteria, comments) and present them.

## Error Handling

- **`gh` not logged in**: Tell the user to run `gh auth login`.
- **`az` not logged in**: Tell the user to run `az login` and `az devops configure --defaults organization=<org> project=<project>`.
- **No results**: Say "No items found matching that filter" and suggest alternative queries.
- **Network/API errors**: Show the error and suggest retrying.

## Rules

- Auto-detect the platform from `git remote get-url origin` — never hardcode org/project/repo
- Default to current sprint (AzDO) or all open issues (GitHub) if no filter is given
- Keep output concise — never dump raw JSON to the user
- This skill is read-only — do not modify any items
- Match the language of the items when responding
