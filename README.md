# Crypto Skills for Python Devs

Skills for Claude and other LLMs/Agents to help users and agents quickly learn and use the Ape stack (Ape Framework, Silverback, and associated SDKs and tools) to build high-quality Python projects and integrations for blockchain networks.

## Available Skills

### Development
- **new-project** - Architect new Python-based blockchain projects using the Ape framework
- **spec-interview** - Clarify under-specified project requirements through guided Q&A
- **python-packages** - Best practices for developing and packaging Python projects

### Ape Framework
- **protocol-design** - Design smart contract protocols using Ape for development, testing, scripting, and deployment
- **writing-plugins** - Create plugins to extend Ape Framework with new compilers, providers, and more
- **migrate-from-brownie** - Migrate Python smart-contract projects from the deprecated Brownie framework to Ape, using a community codemod plus AI-assisted manual cleanup

### Protocols
- **writing-sdks** - Build Python SDKs for blockchain protocols using Ape Framework
- **uniswap** - Trade and interact with Uniswap DEX using Python and the Ape framework

### Silverback
- **writing-bots** - Write bots to continuously listen and respond to events on blockchain networks

## How to Use Skills

### In Claude.ai or Claude Desktop

1. Go to Settings > Skills
2. Click "Add skill repository"
3. Enter: `https://github.com/ApeWorX/skills`
4. Enable the skills you want to use

Once enabled, you can invoke skills by mentioning what you want to do:
- "Help me write a Silverback bot that..."
- "I need to get Uniswap prices for..."
- "Help me create a new Ape plugin for..."
- "I want to start a new blockchain project"

### In Claude Code

When working in a project, you can invoke skills by asking Claude to help with specific tasks:

```
Use the Silverback skill to help me write a bot that monitors USDC transfers
```

```
Use the Uniswap skill to help me swap tokens on Ethereum mainnet
```

```
Help me architect a new Python blockchain project using the new-project skill
```

Skills are automatically discovered and loaded when you ask Claude for help with tasks they cover.

### Best Practices

- **Be specific** about what you want to accomplish
- **Mention requirements** like which network, which tokens, what events to monitor
- **Combine skills** - For example, use Silverback to create a trading bot, then integrate Uniswap for DEX interactions
- **Let Claude fetch docs** - Skills automatically retrieve the latest documentation, so you always get current information

## Marketplace Configuration

Skills are configured in `.claude-plugin/marketplace.json` for discovery and installation.

See individual skill documentation in `./development/`, `./ape/`, `./protocols/`, and `./silverback/` directories.

## License

Apache 2.0 License
