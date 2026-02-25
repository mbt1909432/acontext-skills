# acontext-skills

Agent skills for building AI chatbots with Acontext SDK integration.

## Available Skills

### acontext-chatbot-integration

Build AI chatbots with Acontext integration for persistent storage, file operations, and Python sandbox execution.

**Features:**
- Session management with conversation history storage
- Disk tools (read/write/list/grep files)
- Python sandbox execution for data analysis
- Artifact storage with `disk::` protocol
- Token-aware context compression

**Install:**

```bash
# 安装所有 skills
npx skills add mbt1909432/acontext-skills

# 只安装这个 skill
npx skills add mbt1909432/acontext-skills --skill acontext-chatbot-integration

# 或者直接指定路径
npx skills add https://github.com/mbt1909432/acontext-skills/tree/main/acontext-chatbot-integration
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

See [acontext-chatbot-integration/SKILL.md](./acontext-chatbot-integration/SKILL.md) for detailed usage instructions.
