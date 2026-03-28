---
name: create-pr
description: >
    Create pull requests (GitHub or Azure DevOps) with auto-generated title and description.
    Use when user wants to create a PR, open a PR, or push their work for review.
    Triggers on: "create PR", "write PR", "push for review", "open PR", "send to review".
allowed-tools: Bash, Read, Grep, Glob
---

# PR Creation

Create pull requests with auto-generated title and description from the branch diff. Works with GitHub and Azure DevOps.

## Workflow

### Step 1: Check for uncommitted changes

```bash
git status --porcelain
```

If there are uncommitted changes, **warn the user** and ask if they want to continue. Do not proceed until confirmed.

### Step 2: Detect the platform and remote

```bash
git remote get-url origin
```

Parse the remote URL to determine the platform and extract config:

- **GitHub**: URL contains `github.com` → use `gh` CLI
  - Extract `owner` and `repo` from the URL
- **Azure DevOps**: URL contains `dev.azure.com` or `visualstudio.com` → use `az repos` CLI
  - Extract `org`, `project`, and `repository` from the URL
  - Azure DevOps remote format: `https://dev.azure.com/{org}/{project}/_git/{repo}`

### Step 3: Detect branches

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

Determine the default target branch (usually `main` or `master`):

```bash
git remote show origin | grep 'HEAD branch'
```

#### GitHub — check for existing PR:

```bash
gh pr view --json number,title,baseRefName,url 2>/dev/null
```

#### Azure DevOps — check for existing PR:

```bash
az repos pr list --source-branch "$BRANCH" --status active \
  --org "$ORG_URL" --project "$PROJECT" --repository "$REPO" \
  --query "[0].{id:pullRequestId, title:title, targetRefName:targetRefName, url:url}" -o json
```

If a PR already exists, show its details and ask the user if they want to update the description. If yes, use the update command instead of create.

### Step 4: Gather diff information

Run these in parallel:

```bash
git log "$TARGET_BRANCH"..HEAD --oneline
git diff "$TARGET_BRANCH"...HEAD --stat
git diff "$TARGET_BRANCH"...HEAD
```

### Step 5: Push branch if needed

```bash
git rev-parse --abbrev-ref @{upstream} 2>/dev/null
```

If this fails (no upstream), push:

```bash
git push -u origin "$BRANCH"
```

If it succeeds, check if local is ahead and push if needed:

```bash
git status -sb
```

### Step 6: Extract ticket number (optional)

Parse the branch name for a ticket/issue number. Common patterns:

- `feat/123-some-description` → ticket `123`
- `fix/PROJ-456-bug-title` → ticket `PROJ-456`
- `chore/update-deps` → no ticket

### Step 7: Generate PR title and description

Analyze all commits and the full diff to generate:

**Title**: Short, descriptive, under 72 characters. Do not include the ticket number in the title.

**Description** using this template:

```markdown
## Summary

- <2-3 bullets: what changed and why>

## Changes

<File-level summary grouped by logical area, e.g.:>

**Frontend**

- `src/components/Foo.tsx` — Added error handling for...

**Backend**

- `src/api/bar.ts` — New endpoint for...

**Tests**

- `tests/foo.spec.ts` — Coverage for...

## Testing

- <What was tested>
- <What to verify manually>

<If ticket number found, append a reference line appropriate to the platform:>
<GitHub: "Closes #<number>" or "Refs #<number>">
<Azure DevOps: "AB#<number>">
```

**Language**: Match the language used in the commit messages.

### Step 8: Present for approval

Show the generated title and description to the user. Wait for approval before creating the PR. Accept edits if the user wants to change anything.

### Step 9: Create or update the PR

#### GitHub — create:

```bash
gh pr create \
  --title "<title>" \
  --body "$(cat <<'EOF'
<description>
EOF
)" \
  --base "$TARGET_BRANCH" \
  --head "$BRANCH"
```

#### GitHub — update existing:

```bash
gh pr edit <number> \
  --title "<title>" \
  --body "$(cat <<'EOF'
<description>
EOF
)"
```

#### Azure DevOps — create (no ticket):

```bash
az repos pr create \
  --title "<title>" \
  --description "$(cat <<'EOF'
<description>
EOF
)" \
  --source-branch "$BRANCH" \
  --target-branch "$TARGET_BRANCH" \
  --org "$ORG_URL" \
  --project "$PROJECT" \
  --repository "$REPO" \
  --open
```

#### Azure DevOps — create (with ticket):

```bash
az repos pr create \
  --title "<title>" \
  --description "$(cat <<'EOF'
<description>
EOF
)" \
  --source-branch "$BRANCH" \
  --target-branch "$TARGET_BRANCH" \
  --work-items "<ticket-number>" \
  --org "$ORG_URL" \
  --project "$PROJECT" \
  --repository "$REPO" \
  --open
```

#### Azure DevOps — update existing:

```bash
az repos pr update \
  --id "<pr-id>" \
  --title "<title>" \
  --description "$(cat <<'EOF'
<description>
EOF
)" \
  --org "$ORG_URL" \
  --project "$PROJECT"
```

### Step 10: Output the result

Show the PR URL to the user.

## Rules

- Never force-push
- Always warn about uncommitted changes before proceeding
- Auto-detect the platform (GitHub vs Azure DevOps) from the remote URL
- Auto-detect org/project/repo — never hardcode them
- Auto-detect existing PRs and offer to update instead of creating a duplicate
- Always use a HEREDOC for the description to preserve formatting
- Always present the PR details for user approval before creating
- Keep the title under 72 characters
- Group file changes by logical area in the description
