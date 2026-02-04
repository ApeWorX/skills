---
name: new-project
description: |
  Architect new Python-based blockchain projects using the Ape framework.
  Use when users want to start a new blockchain project and need guidance on proper project structure, architecture patterns, configuration, and foundational setup.
  Supports SDK development, plugin creation, bot development with Silverback, protocol development, and complex blockchain applications.
  Determines project type and guides users through establishing the correct architectural foundation.
compatibility: Requires uv installed
---

# Overview

Guide users through architecting Python-based blockchain projects using the Ape framework.
This skill helps establish proper project structure, configuration, and architectural patterns before implementation begins.

## Prerequisites

Before using this skill, verify the user has:

- `uv` installed (https://docs.astral.sh/uv)
- Basic understanding of Python and blockchain concepts
- Clear project goals (even if high-level)
  - Use the `development/spec-interview` skill if project is not fully specified
- Familiarity with Ape documentation: https://docs.apeworx.io/ape/stable/userguides/quickstart

## Workflow

### Step 1: Determine Project Type

Ask focused questions to understand what the user wants to build:

**Essential questions:**

- What problem are you solving or what functionality do you need?
- Will this interact with existing protocols or create something new?
- Does this need to monitor blockchain events or execute transactions?
- Is this for a specific protocol, a general tool, or a new application?

Based on responses, categorize the project as one of:

1. **Protocol SDK** - Python library for interacting with an existing protocol (e.g. Uniswap)
2. **Ape Plugin** - Extension adding functionality to the Ape framework (e.g. Trezor hardware wallet)
3. **Blockchain Bot** - Automated system responding to on-chain events (e.g. Trading bot)
4. **Protocol Development** - A completely new smart contract protocol (including scripts and tests)
5. **Blockchain Application** - A complex application integrating with one (or more) blockchain(s)

### Step 2: Gather Project-Specific Context

Based on project type, ask targeted follow-up questions.
Keep questions focused and avoid overwhelming the user.

**For Protocol SDKs:**

- Which protocol/contracts will this SDK interact with?
- What are the main operations users will perform with the SDK?
- Do you have a link to the source code and/or documentation for the protocol?
- Reference the `protocols/writing-sdks` skill for more detailed workflow

**For Ape Plugins:**

- What functionality will this add to Ape? (Plugin vertical e.g. AccountAPI, CompilerAPI, ProviderAPI, etc.)
- Does this integrate with a command-line tool (installed locally) or an external service (requiring an API key)?
- Does it need it's own command-line interface integrated with ape? (e.g. `ape [plugin-name] ...`)
- Reference the `ape/writing-plugins` skill for more detailed workflow

**For Blockchain Bots:**

- Which events should trigger actions?
- What actions should the bot perform, in response to those triggers?
- Does it need to sign and submit messages or transactions?
- Reference the `silverback/writing-bots` skill for detailed bot architecture

**For Protocol Development:**

- Which smart contract language do you want to use? (Solidity, Vyper)
- What does the protocol do?
- Do you need multi-chain deployment support?
- Reference the `ape/protocol-design` skill for more detailed workflow

**For Blockchain Applications:**

- What existing protocol(s) does this application need to integrate with?
- What blockchain operations are needed? (submitting transactions, listening to events)
- What's the deployment strategy? How does it need to be hosted?
- Reference the `apps/system-design` skill for more detailed workflow

### Step 3: Generate Project Architecture

Create the appropriate project structure based on chosen project type.

**Common elements across all projects:**

1. Verify project structure is complete
2. Ensure `pyproject.toml` has required dependencies
3. Ensure README.md includes setup instructions
4. Install project locally with `uv sync`

### Step 4: Provide Context and Next Steps

After generating the architecture and installing the project:

1. **Explain the structure** - Briefly describe key files and their purposes
2. **Identify knowledge gaps** - Note where additional skills or documentation are needed:
3. **Recommend next steps** - Clear guidance on what to implement first

### Step 5: Iterate and Refine

After user reviews the architecture:

- Answer questions about design decisions
- Adjust structure based on feedback
- Add missing configurations or files
- Provide additional references as needed

## Key Principles

**Stay high-level**: Focus on correct architecture, not implementation details
**Fetch documentation**: Always get current Ape/Silverback docs before proceeding
**Reference other skills**: Point to specialized skills for detailed workflows
**Adapt to skill level**: Adjust explanations based on user's Python/blockchain experience
**Validate understanding**: Confirm project requirements before generating structure

## Common Pitfalls

- Don't start coding implementation before architecture is confirmed
- Don't assume package versions - check current stable releases
- Don't create overly complex structures for simple projects
- Don't skip documentation fetching - APIs and best practices change
- Don't forget to specify where additional skills/docs are needed
