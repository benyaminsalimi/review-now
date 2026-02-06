# My AI Tools

A collection of AI development tools, skills, and prompts for various AI platforms and frameworks.

## Structure

```
my-ai-tools/
├── claude-code-plugins/    # Claude Code plugins and skills
├── skills/                 # Standalone AI skills and agents
└── prompts/               # Reusable prompts and templates
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

## Prompts

_Coming soon_

## Contributing

Feel free to submit PRs with new tools, skills, or prompts!

## License

MIT License - see [LICENSE](LICENSE) file for details.
