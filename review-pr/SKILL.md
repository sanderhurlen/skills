---
name: review-pr
description: >
    Review pull requests (GitHub or Azure DevOps) with inline comments.
    Use when the user asks to review a PR, check a PR, look at a PR,
    or provides a PR number/URL for review.
    Triggers on: "review PR", "check PR 4565", "review pull request", any PR number reference.
argument-hint: '[pr-number]'
allowed-tools: Bash, Read, Grep, Glob
---

# PR Review

Review pull requests and post inline comments. Works with GitHub and Azure DevOps.

## Workflow

### Step 1: Detect the platform

```bash
git remote get-url origin
```

Parse the URL to determine the platform and extract config:

- **GitHub**: URL contains `github.com` → use `gh` CLI. Extract `owner/repo`.
- **Azure DevOps**: URL contains `dev.azure.com` → use `az repos` CLI. Extract `ORG_URL`, `PROJECT`, `REPO`.

### Step 2: Resolve the PR number

Extract the PR number from `$ARGUMENTS`. Accept:

- Just a number: `42`
- With prefix: `PR 42`, `#42`
- A full URL (extract the number from it)

**If no PR number is given**, auto-detect from the current branch:

#### GitHub:

```bash
gh pr view --json number,title,url 2>/dev/null
```

#### Azure DevOps:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
az repos pr list --source-branch "$CURRENT_BRANCH" --status active --top 1 \
  --org "$ORG_URL" --project "$PROJECT" --repository "$REPO" \
  --query "[0].pullRequestId" -o tsv
```

If auto-detect finds a PR, use it and tell the user which PR was found. If none is found, list open PRs for the user to choose from:

#### GitHub:

```bash
gh pr list --limit 10
```

#### Azure DevOps:

```bash
az repos pr list --status active --reviewer "@Me" --top 10 \
  --org "$ORG_URL" --project "$PROJECT" --repository "$REPO" -o table
```

### Step 3: Fetch PR details

Run these in parallel:

#### GitHub:

```bash
# PR metadata
gh pr view <PR_NUMBER> --json number,title,body,author,baseRefName,headRefName,state,isDraft,reviewRequests,labels,additions,deletions,changedFiles

# Full diff
gh pr diff <PR_NUMBER>

# Existing comments
gh pr view <PR_NUMBER> --json comments,reviews

# Linked issues (from PR body — extract "Closes #N" / "Fixes #N" patterns)
```

#### Azure DevOps:

```bash
# PR metadata
az repos pr show --id <PR_ID> \
  --query "{id:pullRequestId, title:title, description:description, createdBy:createdBy.displayName, sourceRefName:sourceRefName, targetRefName:targetRefName, status:status, isDraft:isDraft, reviewers:reviewers[].{name:displayName,vote:vote}}" \
  --org "$ORG_URL" --project "$PROJECT" -o json

# Diff via git
SOURCE=$(az repos pr show --id <PR_ID> --query "sourceRefName" -o tsv | sed 's|refs/heads/||')
TARGET=$(az repos pr show --id <PR_ID> --query "targetRefName" -o tsv | sed 's|refs/heads/||')
git fetch origin "$TARGET" "$SOURCE" 2>/dev/null
git diff "origin/$TARGET...origin/$SOURCE"

# Existing comments
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project="$PROJECT" repositoryId="$REPO" pullRequestId=<PR_ID> \
  --http-method GET --api-version 7.1 --org "$ORG_URL"

# Linked work items
az repos pr work-item list --id <PR_ID> \
  --query "[].{id:id, title:fields.\"System.Title\", state:fields.\"System.State\", type:fields.\"System.WorkItemType\"}" \
  --org "$ORG_URL" -o table
```

### Step 4: Analyze the diff

Read the full diff carefully. For each file, consider:

- **Correctness** — Logic errors, off-by-one, null/undefined handling
- **Security** — XSS, injection, exposed secrets, unsafe patterns
- **Performance** — Unnecessary re-renders, N+1 patterns, missing memoization
- **Consistency** — Does it follow existing patterns in the codebase?
- **Types** — Missing or incorrect TypeScript/type annotations
- **Tests** — Are changes covered? Should they be?
- **Edge cases** — What could go wrong?

If you need to understand existing patterns, read relevant files in the repo.

### Step 5: Present the review

Format your review as:

```
## PR <number>: <title>
**Author**: <name> | **Files**: <count> | **+<additions>/-<deletions>**
**Work items / linked issues**: <linked items or "None">

### Summary
<2-3 sentences: what this PR does and why>

### Assessment
<Overall verdict: looks good / needs minor changes / needs discussion>

### Comments
<Numbered list of specific, actionable feedback>

For each comment:
1. **file:line** — What the issue is and why it matters
```

### Step 6: Post comments automatically

Do NOT ask the user what to do — post comments immediately after analysis.

**NEVER set a vote/approve/reject** — that is always done by a human.

Skip nit-picks (formatting, style) unless egregiously wrong.

Every comment MUST be prefixed with `🤖 **Claude Code Review** —`.

#### GitHub — general comment:

```bash
gh pr comment <PR_NUMBER> --body "🤖 **Claude Code Review** — <comment>"
```

#### GitHub — review with inline comments:

```bash
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/reviews \
  --method POST \
  --field event=COMMENT \
  --field body="🤖 **Claude Code Review** — Overall summary" \
  --field "comments[][path]=<file>" \
  --field "comments[][position]=<diff-hunk-position>" \
  --field "comments[][body]=🤖 **Claude Code Review** — <comment>"
```

#### Azure DevOps — general comment:

```bash
cat > /tmp/pr-comment.json << 'COMMENT'
{"comments":[{"parentCommentId":0,"content":"🤖 **Claude Code Review** — <comment>","commentType":1}],"status":1}
COMMENT
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project="$PROJECT" repositoryId="$REPO" pullRequestId=<PR_ID> \
  --http-method POST --api-version 7.1 --in-file /tmp/pr-comment.json --org "$ORG_URL"
rm -f /tmp/pr-comment.json
```

#### Azure DevOps — file-specific comment:

```bash
cat > /tmp/pr-comment.json << 'COMMENT'
{
  "comments": [{"parentCommentId": 0, "content": "🤖 **Claude Code Review** — <comment>", "commentType": 1}],
  "status": 1,
  "threadContext": {
    "filePath": "/<path-from-repo-root>",
    "rightFileStart": {"line": <line>, "offset": 1},
    "rightFileEnd": {"line": <line>, "offset": 1}
  }
}
COMMENT
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project="$PROJECT" repositoryId="$REPO" pullRequestId=<PR_ID> \
  --http-method POST --api-version 7.1 --in-file /tmp/pr-comment.json --org "$ORG_URL"
rm -f /tmp/pr-comment.json
```

## Rules

- Auto-detect the platform from `git remote get-url origin` — never hardcode org/project/repo
- Be specific and actionable — "This could be null on line 42" not "consider error handling"
- Reference existing patterns in the codebase when suggesting alternatives
- Don't nitpick formatting or style unless it's egregiously inconsistent
- Praise good patterns — not everything needs to be a criticism
- If the PR is a draft, note that and adjust expectations
- Match the language used in the PR description and commit messages
- Quote shell variables to prevent word-splitting with branch names containing special characters
