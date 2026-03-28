---
name: enforce-pnpm
description: Set up Claude Code hooks to block npm and yarn commands before they execute, enforcing pnpm usage. Use when user wants to enforce pnpm, prevent npm/yarn usage, or add a pnpm guardrail in Claude Code.
---

# Enforce pnpm

Sets up a PreToolUse hook that intercepts and blocks `npm` and `yarn` commands before Claude executes them, enforcing `pnpm` across the project.

## What Gets Blocked

- `npm install`, `npm run`, `npm ci`, `npm add`, etc.
- `yarn`, `yarn add`, `yarn install`, etc.

When blocked, Claude sees a message telling it to use `pnpm` instead.

## Steps

### 1. Ask scope

Ask the user: install for **this project only** (`.claude/settings.json`) or **all projects** (`~/.claude/settings.json`)?

### 2. Copy the hook script

The bundled script is at: [scripts/block-npm-yarn.sh](scripts/block-npm-yarn.sh)

Copy it to the target location based on scope:

- **Project**: `.claude/hooks/block-npm-yarn.sh`
- **Global**: `~/.claude/hooks/block-npm-yarn.sh`

Make it executable with `chmod +x`.

### 3. Add hook to settings

Add to the appropriate settings file:

**Project** (`.claude/settings.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-npm-yarn.sh"
          }
        ]
      }
    ]
  }
}
```

**Global** (`~/.claude/settings.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-npm-yarn.sh"
          }
        ]
      }
    ]
  }
}
```

If the settings file already exists, merge the hook into the existing `hooks.PreToolUse` array — don't overwrite other settings.

### 4. Verify

Run a quick test:

```bash
echo '{"tool_input":{"command":"npm install"}}' | <path-to-script>
```

Should exit with code 2 and print a BLOCKED message to stderr.
