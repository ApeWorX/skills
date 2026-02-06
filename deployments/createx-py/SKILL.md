---
name: createx-py
description: Add deterministic multi-chain deployments to your smart contract project using the CreateX factory and the createx-py SDK.
compatibility: Requires Python, ape installed (with configured `ape accounts`), and an ape project with compilable contracts
---

This skill describes when and how to use the `createx-py` SDK to deploy smart contracts to deterministic addresses
across 170+ supported chains using the CreateX factory contract.

The user provides which contract(s) they want to deploy, which account to deploy from,
and any preferences for salt values, redeploy protection, or vanity address mining.

## Using This Skill

**CRITICAL**: Before writing any code or running any commands with this SDK, you MUST:
1. Use `web_fetch` to retrieve the latest documentation from https://github.com/ApeWorX/createx-py/blob/main/README.md
2. Use `web_fetch` to retrieve the latest Ape documentation from https://docs.apeworx.io/ape/stable
3. Specifically fetch relevant pages like:
   - Setting up an account with Ape: https://docs.apeworx.io/ape/stable/userguides/accounts#live-network-accounts
   - CreateX redeploy protection: https://github.com/pcaversaccio/createx/tree/main?tab=readme-ov-file#permissioned-deploy-protection-and-cross-chain-redeploy-protection

**DO NOT** rely on general knowledge about CreateX or Ape - always fetch the current documentation first to ensure accuracy.

## Deploying a Contract

Before deploying, understand the user's requirements:
- **Which contract**: The contract type from their Ape project to deploy (e.g. `MyContract`)
- **Which deployer account**: The Ape account alias to sign and pay for the deployment
- **Redeploy protection**: Whether to use `msg.sender` and/or `chainid`-based redeploy protection (enabled by default)
- **Salt**: An optional custom salt value for deterministic address generation
- **Constructor arguments**: Any arguments the contract constructor requires

### CLI Usage

The primary way to use createx-py is through the `createx` CLI command within an Ape project.

**Basic deployment** (with default redeploy protection):
```sh
$ createx deploy MyContract --deployer my-wallet --salt my-salt
```

**Deployment without redeploy protection** (same address regardless of deployer or chain):
```sh
$ createx deploy MyContract --deployer my-wallet --no-redeploy-protection --salt my-salt
```

### Library Usage

For programmatic deployments (e.g. in deployment scripts):
```python
from createx import CreateX

my_contract = createx.deploy(project.MyContract, salt="something", sender=me)
```

## Mining Vanity Addresses

The SDK includes a utility for discovering salts that produce addresses with specific patterns,
such as leading zeros. This is useful for generating memorable or gas-efficient contract addresses.

**Mine an address with leading zeros:**
```sh
$ createx mine dep@v1:MyContract --deployer my-wallet --leading-zeros 1
```

Once a salt is found, verify the predicted address and then deploy with it:
```sh
$ createx address dep@v1:MyContract --deployer my-wallet --salt <found-salt>
$ createx deploy dep@v1:MyContract --deployer my-wallet --salt <found-salt>
```

The same salt produces the same address across multiple chains,
so you can mine once and deploy to the same address everywhere.

## Multi-Chain Deployments

A key benefit of CreateX is deploying to the same address on multiple chains.
To achieve this:
1. Choose or mine a salt value
2. Verify the predicted address using `createx address`
3. Deploy on each target chain using the same salt, deployer, and contract bytecode

**CRITICAL**: The contract bytecode must be identical across chains for the address to match.
If constructor arguments differ per chain, the resulting address will differ.

## Managing Risk

Deterministic deployments are powerful but require care:
- Always verify the predicted address with `createx address` before deploying
- Test deployments on a testnet first to confirm the address and behavior
- Be aware that redeploy protection settings affect address derivation;
  changing protection settings changes the deployed address
- When deploying to multiple chains, deploy to a testnet first to validate everything works as expected
- Keep track of your salt values, as losing them means losing the ability to predict or redeploy to the same address
