# Contributing to Miget Agent Skills

We welcome contributions from the community! Whether it's a bug fix, a new skill, or an improvement to an existing one, your input helps make Miget better for everyone.

## Getting Started

1. Fork this repository
2. Clone your fork locally
3. Create a new branch for your changes

## Project Structure

```
skills/
  miget-api/
    SKILL.md        # Miget API skill definition
```

Each skill lives in its own directory under `skills/` and contains a single `SKILL.md` file following the [Agent Skills Open Standard](https://agentskills.io/home).

## Writing a Skill

Every `SKILL.md` must include YAML frontmatter with `name` and `description` fields:

```yaml
---
name: your-skill-name
description: A concise description of what the skill does and when to use it.
---
```

The body should provide clear, structured instructions that an AI agent can follow to accomplish tasks.

## Submitting Changes

1. Make sure your changes follow the existing style and conventions
2. Update the `CHANGELOG.md` with a summary of your changes
3. Submit a pull request with a clear description of what you changed and why

## Reporting Issues

Found a bug or have a suggestion? [Open an issue](https://github.com/migetapp/agent-skills/issues) and include as much detail as possible:

- What you expected to happen
- What actually happened
- Steps to reproduce (if applicable)

## Code of Conduct

Please read our [Code of Conduct](CODE_OF_CONDUCT.md) before contributing. We are committed to providing a welcoming and inclusive experience for everyone.
