# AI Agent Skills

A collection of practical AI agent skills for OpenClaw and other agent systems.

## Skill Organization

Each skill is organized in its own subdirectory:

```
skill-name/
 丕 SKILL.md       # Main skill definition
├── examples/      # Optional examples
├── tests/         # Optional tests
└── README.md      # Optional documentation
```

## Installation

To use these skills in your AI agent:

1. **Identify your agent workspace path:**
   - OpenClaw: `~/.openclaw/workspace/skills/`
   - Other agents: Check your agent's documentation

2. **Create the skill directory:**
   ```bash
   mkdir -p ~/.openclaw/workspace/skills/skill-name
   ```

3. **Copy the SKILL.md file:**
   ```bash
   curl -s https://raw.githubusercontent.com/mwaeckerlin/ai-agent-skills/main/skill-name/SKILL.md -o ~/.openclaw/workspace/skills/skill-name/SKILL.md
   ```

4. **Verify installation:**
   ```bash
   openclaw skills list | grep skill-name
   ```

## Available Skills

- [`language`](./language/SKILL.md) - Language handling conventions for AI agents
- [`github-ticket`](./github-ticket/SKILL.md) - Automated GitHub issue management with Copilot

## Contributing

To add a new skill:

1. Create a new directory `skill-name/`
2. Add your `SKILL.md` following the OpenClaw Skill specification
3. Optionally add examples and tests
4. Update this README with the new skill
5. Submit a pull request

## License

MIT License - See LICENSE file for details
