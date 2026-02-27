# acontext-skills

Agent skills for building AI chatbots with Acontext SDK integration.

## Available Skills

### acontext-agent-integration

Build AI agents with Acontext integration for persistent storage, file operations, and Python sandbox execution.

**Features:**
- Session management with conversation history storage
- Disk tools (read/write/list/grep files)
- Python sandbox execution for data analysis
- Artifact storage with `disk::` protocol
- Token-aware context compression

**Install:**

```bash
# Install all skills
npx skills add mbt1909432/acontext-skills

# Install only this skill
npx skills add mbt1909432/acontext-skills --skill acontext-agent-integration

# Or specify the path directly
npx skills add https://github.com/mbt1909432/acontext-skills/tree/main/acontext-agent-integration
```

## Usage

After installation, the skill will be available in your AI agent. Use it when:
- Building chatbots with conversation history storage
- Implementing file read/write/list/grep tools
- Adding Python code execution for data analysis
- Creating figure generation pipelines
- Managing artifacts with disk:: protocol
- Implementing token-aware context compression

## Documentation

See [acontext-agent-integration/SKILL.md](./acontext-agent-integration/SKILL.md) for detailed usage instructions.
