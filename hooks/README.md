# Hooks

Claude Code hooks for automating workflows and adding custom behavior.

## What are Hooks?

Hooks are shell commands that execute automatically in response to Claude Code events:
- **PreToolUse**: Before Claude calls a tool
- **PostToolUse**: After a tool completes
- **UserPromptSubmit**: When user submits a message
- **Stop**: When Claude's response completes

## Coming Soon

This directory will contain reusable hooks for:
- Pre-commit validation
- Automatic linting and formatting
- Security checks
- Custom workflows
- And more...

## Hook Examples

### Pre-Commit Hook
Automatically run linters before commits:
```json
{
  "hooks": {
    "PreToolUse": {
      "tool": "Bash",
      "pattern": "git commit",
      "command": "npm run lint && npm test"
    }
  }
}
```

### Security Check Hook
Scan for secrets before committing:
```json
{
  "hooks": {
    "PreToolUse": {
      "tool": "Write",
      "command": "echo 'Checking for secrets...' && git diff --cached | grep -i 'password\\|secret\\|key'"
    }
  }
}
```

## Documentation

For more information on hooks:
- [Claude Code Hooks Documentation](https://code.claude.com/docs/en/hooks.md)
