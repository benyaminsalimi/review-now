# My AI Tools

A collection of AI development tools, plugins, skills, and hooks for Claude Code and other AI platforms.

## Structure

```
my-ai-tools/
├── claude-code-plugins/    # Claude Code plugins
├── skills/                 # Standalone AI skills and agents
└── hooks/                  # Claude Code hooks for automation
```

## Claude Code Plugins

### code-review

A multi-agent code review plugin for Claude Code that provides comprehensive PR and uncommitted changes review.

**Installation:**
```bash
cd claude-code-plugins/code-review
claude plugin marketplace add ./
claude plugin install code-review@local-dev
```

**Usage:**
```bash
# Review uncommitted changes
/code_review uncommitted

# Compare two branches
/code_review feature-branch main
```

**Features:**
- Multi-agent review (security + general code quality)
- Validation layer to filter false positives
- Interactive fix walkthrough for uncommitted changes
- Support for custom guidelines via REVIEW_GUIDELINES.md
- AGENTS.md support for directory-scoped rules

See [code-review/README.md](claude-code-plugins/code-review/README.md) for detailed documentation.

## Skills

_Coming soon_

Standalone AI skills and agent configurations that can be used across different platforms.

## Hooks

_Coming soon_

Claude Code hooks for:
- Pre-commit validation
- Automatic linting and formatting
- Security checks
- Custom workflow automation

See [hooks/README.md](hooks/README.md) for examples and documentation.

## Contributing

Feel free to submit PRs with new tools, skills, or prompts!

## License

MIT License - see [LICENSE](LICENSE) file for details.
