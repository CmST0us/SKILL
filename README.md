# Claude Code Skills Collection

A collection of reusable skills for Claude Code (claude.ai/code).

## Available Skills

| Skill | Description |
|-------|-------------|
| [embedded-swift](./embedded-swift/) | Embedded Swift 开发规范检查清单 |

## How to Use

### Method 1: Copy to Project

Copy the skill folder to your project's `.claude/skills/` directory:

```bash
# Create skills directory if not exists
mkdir -p .claude/skills

# Copy the skill
cp -r embedded-swift/embedded-swift.md .claude/skills/
```

Then add to your `.claude/commands.json`:

```json
{
  "embedded-swift": {
    "description": "Embedded Swift 开发规范检查",
    "skill": "embedded-swift"
  }
}
```

### Method 2: Reference Directly

You can also reference these skills directly in your conversations with Claude Code by asking it to read the skill file from this repository.

## Contributing

Feel free to add new skills by creating a pull request. Each skill should:

1. Have its own directory
2. Include a markdown file with the skill content
3. Be documented in this README

## License

MIT
