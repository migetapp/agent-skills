# Miget Agent Skills

[![License](https://img.shields.io/badge/License-BSD_3--Clause-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.9-green.svg)](CHANGELOG.md)

Agent skills for [Miget](https://miget.com) - a Kubernetes-based Platform-as-a-Service (PaaS) for deploying and managing applications, databases, and services in the cloud.

## Getting Started

This repository provides agent skills following the [Agent Skills Open Standard](https://agentskills.io/home). Agent skills give AI coding assistants (Claude Code, Cursor, Windsurf, etc.) the context they need to work with the Miget API effectively.

### Install the skills

```bash
npx skills add migetapp/agent-skills
```

### Available skills

- **miget-api** - Deploy and manage apps, databases, buckets, and services on Miget PaaS. Covers authentication, resource provisioning, deployments, addons, domains, environment variables, and all API endpoints.

### Authentication

Before using the skill, set up your Miget API token:

1. Sign in at [app.miget.com](https://app.miget.com)
2. Go to **My Account > [API Tokens](https://app.miget.com/my_account#api_tokens)**
3. Create a new token (it starts with `miget_live_`)
4. Export it in your shell:

```bash
export MIGET_API_TOKEN="miget_live_xxxxxxxxxxxxx"
```

The agent will automatically detect the token from your environment. For persistent setup, add the export to your `~/.zshrc` or `~/.bashrc`.

### Usage

Once installed, your AI coding assistant will automatically discover the skill and use it when working with Miget. No additional configuration is needed.

### Prompt examples

- "Create a new project on Miget and deploy my Node.js app"
- "Set up a PostgreSQL database as an addon for my app"
- "Configure a custom domain for my Miget app"
- "Show me all apps deployed in my workspace"
- "Create an S3-compatible bucket on Miget and get the credentials"
- "Set environment variables for my production app"
- "Deploy my app from a GitHub repository"

## Contributing

We welcome contributions! Please read our [Contributing Guide](CONTRIBUTING.md) and [Code of Conduct](CODE_OF_CONDUCT.md) before getting started.

## License

This project is licensed under the [BSD 3-Clause License](LICENSE).
