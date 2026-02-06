---
name: smart-contracts
description: |
  General smart contract development principles, design patterns, and security practices.
  Use when users are writing or reviewing smart contract code and need guidance on how to write
  contracts that are secure, readable, and maintainable. Covers architecture, security, testing,
  state management, error handling, event design, access control, and more.
  Language-agnostic with Vyper recommended as default. Works alongside protocol-design for
  project-level guidance, or standalone when working on contracts in an existing project.
---

# Overview

Guide users through writing smart contracts that are secure, readable, and maintainable.
This skill covers the principles and patterns that apply at the **code level** — how to structure
individual contracts, manage state, handle errors, design events, control access, and test
effectively. It is language-agnostic but recommends Vyper as the default choice.

This skill complements the `ape/protocol-design` skill, which covers project-level concerns
(project setup, deployment scripts, automation). Use this skill when the focus is on
**how to write the contracts themselves**.

## When to Use This Skill

- Writing new smart contracts in an existing or new project
- Reviewing contract code for security and design issues
- Deciding between architectural approaches (upgradeability, modularity, etc.)
- Designing the public API, events, errors, and access control for a contract
- Writing or improving contract-level tests
- Evaluating whether to use a dependency or write custom code

## Prerequisites

- Basic understanding of how blockchains and smart contracts work
  - What a transaction is, what gas is, what storage vs. memory means
- A smart contract project already initialized (or use `ape/protocol-design` to set one up)
- A compiler plugin installed (`ape-vyper` or `ape-solidity`) — these handle compiler installation

## Language Choice

This skill is **language-agnostic** — the principles apply to both Vyper and Solidity.
However, if the user has no existing preference, **recommend Vyper** for these reasons:

- Vyper's compiler backend produces efficient bytecode while maintaining compiler-level safety
  guarantees, reducing the need for manual gas optimization or inline assembly
- Vyper's simpler syntax makes contracts easier to read and audit
- Vyper enforces bounds on dynamic arrays, which encourages better state design
- Vyper has built-in SafeMath (overflow/underflow protection) that cannot be bypassed
- Vyper's module system allows separating concerns without the complexity of inheritance chains

**When Solidity may be the better choice:**

- The project needs metaprogramming features Vyper doesn't support (e.g. generics, complex inheritance)
- The team or available auditors are significantly more experienced with Solidity
- The project depends heavily on Solidity-only tooling or libraries
- An existing codebase is already written in Solidity

**Compiler version policy:**

- For new projects, use the latest stable compiler version
- Avoid brand-new major or minor releases that haven't been battle-tested in production yet
- Lock the compiler version when the protocol is ready for audit — do not change it between
  audit and deployment
- The compiler plugin (`ape-vyper` or `ape-solidity`) handles installing the compiler itself;
  users do not need to install Vyper or Solidity separately

**On inline assembly and raw opcodes:**

- Avoid inline assembly and raw opcodes in nearly all circumstances
- Using them bypasses compiler-level safety guarantees (bounds checking, overflow protection,
  reentrancy guards, etc.), which is one of the primary reasons to use a high-level language
- If you find yourself reaching for assembly, first check whether the compiler can produce
  equivalent efficient code — modern Vyper and Solidity compilers are quite good at optimization
- The rare exception is when an opcode has no high-level equivalent and is strictly necessary,
  but this should be clearly documented and isolated

## Contract Architecture

### Design by Areas of Concern

Structure contract systems around **areas of concern** rather than splitting logic into many
small, fine-grained contracts. This approach:

- Produces fewer contract types overall, making the system easier to understand and audit
- Keeps related logic together, reducing cross-contract complexity
- Individual contracts may be larger (sometimes approaching the 24KB size limit), but each one
  is self-contained and focused on a clear responsibility

**Anti-pattern to avoid:** Systems with dozens or hundreds of contract files arranged in
complex dependency graphs. These are extremely difficult to audit, reason about, and maintain.
If your system has more than a handful of core contract types, consider whether the architecture
can be simplified.

### When to Split Contracts

Split into separate contracts when:

- Two pieces of logic genuinely serve different roles in the system (e.g. a vault and a strategy)
- You need independent upgradeability of a specific component (rare — see below)
- You're approaching the 24KB bytecode size limit and the contract has natural seams to split along
- Different contracts need to be deployed or governed independently

Do **not** split contracts just because a file is "getting long." A single well-organized contract
is better than two tightly-coupled contracts that constantly call each other.

### Upgradeability

**Default to immutable contracts.** Upgradeability introduces significant complexity and risk:

- It adds a layer of indirection that makes the system harder to reason about
- Storage layout must be carefully managed across upgrades to avoid corruption
- It creates a privileged upgrade role that becomes an attack vector
- It makes it harder for users to trust what code is actually running

**Use upgradeable proxies only when:**

- You must maintain control of assets in a single contract address while changing the rules
  (e.g. a multisig wallet that needs to evolve its signing logic)
- There is a genuine, specific need identified during design — not "we might need it later"

**If you must use upgradeability:**

- Use a well-known, widely-audited standard (e.g. UUPS, Transparent Proxy from OpenZeppelin)
- Never implement a custom proxy pattern — the attack surface is too large
- Be extremely careful about storage layout across upgrades
- Consider adding a timelock to the upgrade mechanism so users can exit before changes take effect

**Patterns to avoid:**

- **Diamond pattern (EIP-2535):** Too complex for most use cases. It has all the concerns of
  upgradeability plus the risk of comingled storage updates across multiple facet contracts.
  Only consider it if you have an exceptionally strong reason and deep expertise.
- **On-chain library patterns (Solidity):** Avoid. The interface between a contract and a
  library is compiler-version-specific, creating a worse version of the upgradeability problem.
  If the library's compiler version drifts from the calling contract's, things break silently.

### Modules and Code Reuse

**In Vyper:** Use the module system to separate concerns while keeping the contract type count
low. Modules let you extract reusable logic (e.g. access control, math helpers) into separate
files that get compiled into the importing contract — no cross-contract calls, no library
linking, no proxy indirection.

**In Solidity:** Prefer abstract contracts and internal libraries over deployed (external)
libraries. Inheritance is the primary reuse mechanism, but keep inheritance chains shallow
and linear — deep diamond-shaped inheritance graphs are a frequent source of bugs and confusion.

## Security

### Core Philosophy

The most effective way to reduce security issues is to **keep things small and understandable**.
A contract that is easy to read is easy to audit, and a system with fewer moving parts has
fewer places for bugs to hide. Complexity is the enemy of security.

Deploy systems in stages, testing each assumption along the way, to reduce the risk surface
of the final production deployment.

### Security Patterns

The following patterns range from **mandatory** (apply always) to **situational** (apply when
the design calls for it). Understanding all of them is important, even if not every contract
needs every pattern.

#### Checks-Effects-Interactions (Mandatory)

Always structure state-changing functions in this order:

1. **Checks** — Validate all inputs and preconditions (revert early if anything is wrong)
2. **Effects** — Update all contract state
3. **Interactions** — Make external calls (transfers, calls to other contracts)

This ordering prevents reentrancy attacks by ensuring state is finalized before any external
call can re-enter the contract. This is the single most important pattern in smart contract
security.

```
# Vyper example
@external
def withdraw(amount: uint256):
    # 1. Checks
    assert self.balances[msg.sender] >= amount, "Insufficient balance"

    # 2. Effects
    self.balances[msg.sender] -= amount

    # 3. Interactions
    send(msg.sender, amount)
```

```
// Solidity example
function withdraw(uint256 amount) external {
    // 1. Checks
    require(balances[msg.sender] >= amount, "Insufficient balance");

    // 2. Effects
    balances[msg.sender] -= amount;

    // 3. Interactions
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}
```

#### Reentrancy Guards (Mandatory When Checks-Effects-Interactions Is Insufficient)

When function logic is complex enough that the checks-effects-interactions ordering cannot
be cleanly maintained, add an explicit reentrancy guard. Vyper has a built-in `@nonreentrant`
decorator. In Solidity, use OpenZeppelin's `ReentrancyGuard`.

Apply reentrancy guards to any function that:
- Makes external calls AND modifies state
- Is part of a multi-step process where intermediate state could be exploited

#### Input Validation (Mandatory at External Boundaries)

Always validate inputs to externally-accessible functions (`@external` in Vyper, `external`/`public`
in Solidity). This is the boundary between trusted internal logic and untrusted external callers.

- Check that addresses are not zero (when zero would be invalid)
- Check that amounts are within acceptable ranges
- Check that the caller has the right to perform the action
- Check that the contract is in the right state for the operation

For internal functions, rely on the compiler's built-in protections (SafeMath, bounds checking)
rather than adding redundant checks. Vyper's native SafeMath means underflows and overflows
will revert automatically without explicit checks.

#### External Call Safety (Mandatory)

**Always treat external calls as untrusted** unless you have full control over the target
address and have pre-vetted the implementation, or the target is a widely-used and well-audited
contract (such as major token contracts).

- Never assume an external call will behave as expected
- Always check return values
- Be aware that an external call can trigger re-entrance into your contract
- Be aware that an external call can consume arbitrary gas or revert

#### Pull-Over-Push for Payments (Situational)

When distributing funds to multiple recipients, prefer letting recipients **pull** (withdraw)
their funds rather than **pushing** (sending) to them in a loop. This avoids:

- Denial-of-service if one recipient's receive function reverts
- Gas limit issues with large recipient lists
- Reentrancy risks from multiple external calls in a loop

#### Rate Limiting and Circuit Breakers (Situational)

For high-value protocols, consider:

- **Withdrawal limits** — Cap the amount that can be withdrawn in a time period
- **Pause mechanisms** — An emergency stop that halts critical operations
- **Gradual rollouts** — Start with low limits and increase as confidence grows

These are especially valuable during early deployment when the system is least battle-tested.

### ETH / Native Token Transfers

Sending ETH should be a non-interactive experience. Both compilers have tried to enforce
this, but without a protocol-level change (such as the proposed PAY opcode), it is difficult
to guarantee in practice.

- In Solidity, prefer low-level `call{value: amount}("")` over `transfer()` or `send()`,
  as the 2300 gas stipend from `transfer`/`send` can cause legitimate receive functions to fail
- In Vyper, use `send()` for simple transfers; use `raw_call` when you need more control
- Always check the return value of transfer operations

### Common Vulnerability Classes

Understanding common attack patterns is essential to writing secure contracts. Rather than
reproducing full explanations here, be proficient with these categories and understand how
they manifest in practice:

- **Reentrancy** — External calls that re-enter state-changing functions before state is finalized
- **Oracle manipulation** — Price feeds that can be manipulated within a single transaction
- **Flash loan attacks** — Exploiting logic that assumes token balances are stable within a transaction
- **Front-running / MEV** — Transactions that can be observed and exploited before execution
- **Access control bypasses** — Missing or incorrect permission checks on sensitive functions
- **Integer overflow/underflow** — Arithmetic that wraps (mitigated by modern compilers, but relevant in older code)
- **Denial of service** — Logic that can be blocked by a malicious actor (e.g. reverting receive functions)
- **Storage collision** — Overlapping storage slots in proxy/upgradeable patterns

**Recommended external resources:**

- SWC Registry (Smart Contract Weakness Classification): https://swcregistry.io/
- Rekt News (real-world exploit post-mortems): https://rekt.news/
- Vyper documentation: https://docs.vyperlang.org/
- Solidity documentation: https://docs.soliditylang.org/
- Slither static analyzer: https://github.com/crytic/slither

## State Management

### Storage Layout

- **Avoid manual storage packing** unless the compiler handles it natively. Packing multiple
  values into a single storage slot saves gas but creates significant complexity in reading
  and writing those values, and can introduce subtle bugs. When Vyper ships native storage
  packing, this guidance will change — but until then, prefer clarity over density.
- Solidity does support automatic struct packing, but be aware of the considerations: field
  ordering matters, and changes to packed structs can break storage layout in upgradeable contracts.
- Group related state variables together logically for readability, even without packing.

### Mappings vs. Arrays

- **Use mappings** when the collection has unknown or unbounded scale, or when you need O(1)
  lookup by key. Mappings are the default choice for most on-chain collections.
- **Use arrays** when you need to iterate over elements or when the collection has a known,
  bounded size.
- In Vyper, all dynamic arrays require explicit bounds (e.g. `DynArray[address, 100]`), which
  enforces good hygiene. If you find yourself wanting an unbounded array, that's usually a
  signal that a mapping is the better data structure.
- Be cautious with arrays that grow over time — iterating over large arrays in a transaction
  can hit gas limits.

### Contract Address References

- **Use immutable references** (set at deployment) when the target contract address will never
  change. This is cheaper (no storage read) and makes the dependency explicit.
- **Use storage variables** when the target address genuinely needs to be updatable (e.g. pointing
  to a contract that might be redeployed).
- Keep the dependency graph simple and easy to trace. Complex webs of cross-contract address
  references are a frequent source of control flow bugs — if you can't easily diagram which
  contract points to which, simplify.

### Constants, Immutables, and Configurable State

- **Constants** — Values known at compile time that will never change (e.g. `MAX_SUPPLY = 1000000`).
  Use for protocol-wide invariants.
- **Immutables** — Values set once at deployment and never changed (e.g. token addresses,
  initial parameters). Use for deployment-specific configuration.
  Cheaper to read than storage since they're embedded in bytecode.
- **Configurable state (storage)** — Values that need to change after deployment (e.g. fees,
  thresholds, authorized addresses). Use only when mutability is genuinely required, and
  protect mutations with appropriate access control.

The decision hierarchy: **constant > immutable > storage**. Default to the most restrictive
option and only move to a more flexible one when you have a concrete reason.

## Error Handling

### Error Messages and Custom Errors

- **In Vyper:** Use unique, descriptive revert strings. Custom error types are not yet available,
  so clear string messages are the primary way to trace the source of a revert.
  Keep them concise but specific enough to identify which check failed.
  Example: `assert amount > 0, "deposit: zero amount"` rather than just `"error"`.
- **In Solidity:** Use custom errors (`error InsufficientBalance(uint256 requested, uint256 available)`)
  — they are more gas-efficient than revert strings and can carry structured data that makes
  debugging easier.

### Where to Validate

- **Always validate at external entry points** — These are the trust boundary. Every `@external`
  / `external` / `public` function should check that its inputs and the current state are valid
  before proceeding.
- **Rely on compiler guarantees internally** — For internal functions, the compiler's built-in
  protections (SafeMath, bounds checks) catch arithmetic and indexing errors without explicit checks.
- **On modifiers and decorators:** They can be useful for access control checks that are repeated
  across many functions (e.g. `onlyOwner`), but avoid using them for complex validation logic.
  Modifiers can obscure the flow of a function, making it harder to audit. When in doubt,
  prefer inline checks that are visible in the function body.

### Error Verbosity

Errors should be **verbose enough to trace** but not wasteful:

- Include the function or context name so you can locate the error quickly
- Include relevant values when using custom errors (Solidity)
- Don't write paragraph-length revert strings — each byte costs gas in deployment
- A good rule of thumb: could you find this error in the codebase with a quick search?

## Event Design

### What to Emit

Design events for **meaningful state changes** — actions that external systems (monitoring bots,
indexers, UIs, analytics) would want to know about. Not every internal state update needs an event,
but every action that a user or operator might need to react to should emit one.

Events are also the only practical way to get detailed, transaction-by-transaction change history
over long timeframes on-chain, so consider what data would be useful for historical analysis.

Good candidates for events:

- Transfers, deposits, withdrawals
- Configuration changes (fee updates, parameter changes)
- Role changes (ownership transfers, permission grants)
- Significant protocol state transitions (pausing, unpausing, liquidations)

### Indexed Parameters

Choose indexed parameters based on what users will **specifically want to filter on**.
There is a limit of 3 indexed parameters per event (or 4 for anonymous events).

The most common use case: index `address` fields so that users can subscribe to events
that implicate their address (e.g. `Transfer` events where they are the sender or receiver).

### Event Data

- Include **computed or derived data** in events when it would be expensive or complex for
  consumers to recompute. For example, if a function calculates a fee or a new balance,
  emit those computed values rather than forcing every consumer to replicate the calculation.
- Give consumers enough information to reconstruct the context of the event without needing
  to make additional on-chain calls at that block height.

### Naming

- Name events as **past-tense actions**: `Deposited`, `Withdrawn`, `OwnershipTransferred`,
  `FeeUpdated`. Some conventions use present tense (`Transfer`, `Approval`) — either is
  fine as long as it is consistent within the project.

## Interface and API Design

### Public API Surface

- **Expose public view functions for all critically important state**, especially complex
  internal calculations that would not otherwise be easily visible to external consumers.
  If a user or integrating contract would need to know a value, make it readable on-chain.
- Favor **explicit, descriptive function names** over short or overloaded names. A function
  called `calculatePendingRewards(address)` is far more useful than `calc(address)`.
- Separate **read functions** (views) from **write functions** (state-changing) clearly in
  your contract layout for readability.

### Interface Types

- Define explicit interface files (`.vyi` in Vyper, `IContract.sol` in Solidity) when the
  contract is intended to have **multiple implementations** — i.e. when other contracts will
  interact with it through the interface rather than the concrete type.
- For internal contracts that only have one implementation, an interface file adds overhead
  without benefit. Use it when there's a clear consumer or extension point.

### Function Parameters

- Use the **most specific type** available. If a parameter is a token address, type it as the
  token interface rather than a bare `address` where possible.
- In Vyper, use **default parameters** to create ergonomic APIs — order parameters from most
  commonly used to least commonly used, so callers can omit trailing optional arguments.
- Avoid boolean flag parameters that change behavior (e.g. `transfer(amount, is_fee_exempt)`)
  — they make call sites hard to read. Prefer separate functions or explicit parameter types.
- For functions with many parameters, consider whether a struct/tuple would be clearer
  (Solidity). In Vyper, default parameters often solve this more elegantly.

## Access Control

### Minimize Privileged Roles

Privileged roles are an attack vector. If the privileged account (owner, admin, governance)
is compromised, the attacker gains all the power that role has. Minimize both the **number
of roles** and the **power each role has**.

**Progression of complexity (use the simplest that meets your needs):**

1. **No roles** — The contract is fully autonomous after deployment. This is the most secure
   option when feasible. Prefer this when the contract doesn't need post-deployment configuration.
2. **Owner pattern** — A single address can perform administrative actions. Simple, well-understood,
   and sufficient for many contracts. The owner can be an EOA, a multisig, or a governance contract.
3. **Role-based access control** — Multiple distinct roles with different permissions
   (e.g. `pauser`, `minter`, `admin`). Use only for larger, more complex systems where a
   single owner role is genuinely insufficient.
4. **Timelocks and governance** — Operations are delayed or require voting. Reserved for
   large decentralized finance ecosystems where a consortium of users must agree on changes.
   This is the most complex option and should be justified by the scale and decentralization
   requirements of the system.

### Initializers

Use constructors by default. Use initializer patterns only when constructors don't work:

- **Proxy/upgradeable contracts** — The proxy's constructor runs in the proxy's context, not
  the implementation's, so an initializer is needed to set up the implementation's state.
- **Registry/factory patterns** — Contracts created via `create_copy_of` or `clone` that
  cannot use constructors for initialization.

When using initializers, always include a guard to prevent re-initialization:

```
# Vyper
initialized: bool

@external
def initialize(param: address):
    assert not self.initialized, "Already initialized"
    self.initialized = True
    # ... setup logic
```

## Gas Optimization

### Philosophy

**Always prefer readability and security over gas efficiency.** A security vulnerability
costs thousands of times more than the gas savings from any optimization. Write clear,
correct code first. Optimize only when profiling shows a specific, significant cost.

### Practical Guidelines

- **Storage caching** — Reading from storage (`SLOAD`) is expensive. If you read the same
  storage variable multiple times in a function, cache it in a local (memory) variable.
  This is one of the few optimizations that is almost always worth doing, as the gas savings
  are significant and the code remains clear. Vyper is working on native storage caching in
  the compiler backend, which will make this automatic in the future.

```
# Vyper - storage caching
@external
def process():
    balance: uint256 = self.balance  # Cache once
    # Use `balance` local variable instead of `self.balance` for the rest of the function
```

```
// Solidity - storage caching
function process() external {
    uint256 balance = _balance; // Cache once
    // Use `balance` instead of `_balance` for the rest of the function
}
```

- **Short-circuit evaluation** — Put the cheapest and most likely-to-fail conditions first
  in chains of `assert` / `require` statements, so expensive checks are skipped when possible.
- **Avoid unnecessary storage writes** — If a value hasn't changed, don't write it back to
  storage. The `SSTORE` opcode is one of the most expensive operations.
- **Batch operations** — When a user needs to perform multiple similar actions, provide a batch
  function that does them in a single transaction to amortize the base transaction cost.

### Anti-Patterns to Avoid

- **Premature storage packing** — Manually packing multiple values into a single `uint256`
  saves storage slots but adds complexity to every read and write. The cognitive overhead
  and bug potential outweigh the gas savings in nearly all cases. Wait for native compiler
  support.
- **Sacrificing readability for micro-optimizations** — Tricks like using `unchecked` blocks
  aggressively, bitwise operations for division, or avoiding named variables to "save gas"
  make code harder to audit and more likely to contain bugs.
- **Over-optimizing view functions** — View functions don't cost gas when called off-chain
  (only when called from other contracts on-chain). Don't sacrifice their readability for
  gas savings that only apply in cross-contract calls.

## Testing

### Philosophy

Testing is the primary line of defense against bugs. **Formal verification** can be valuable
for high-value, critical systems, but **property-based and fuzz testing** catch most classes
of bugs with far less effort. Invest in good test coverage as a standard practice; escalate to
formal verification when the stakes justify it.

**Coverage** is a useful tool to identify gaps in testing — places where code paths are not
exercised — but it is **not a guarantee of sufficient testing**. 100% line coverage does not
mean 100% of behaviors are tested. Use coverage as a guide to find where it makes sense to
invest further, not as a completeness metric.

### Test Categories

Organize tests into two tiers (these align with the structure from `ape/protocol-design`):

**Functional tests** (`tests/functional/`) — Test each contract in isolation:

- One test file per contract
- Test each public function with valid inputs and expected outputs
- Test edge cases and boundary values (zero, max values, empty arrays)
- Test that invalid inputs revert with the expected error
- Test access control (unauthorized callers must be rejected)
- Test event emissions (verify that the right events are emitted with the right data)

**Integration tests** (`tests/integration/`) — Test the system as a whole:

- Deploy the full contract system and test realistic user scenarios
- Test multi-contract interactions and state flows
- Test with forked mainnet state for protocols that interact with existing DeFi
  (using Ape's fork provider support)
- Test upgrade paths if using upgradeable contracts

### Testing Patterns

#### Revert Testing

Test that functions revert when they should, and verify the error message:

```python
# Using Ape's testing utilities
import ape

def test_withdraw_insufficient_balance(contract, user):
    with ape.reverts("withdraw: insufficient balance"):
        contract.withdraw(1000, sender=user)
```

#### Event Testing

Verify that state-changing functions emit the correct events:

```python
def test_deposit_emits_event(contract, user):
    receipt = contract.deposit(100, sender=user, value=100)
    events = contract.Deposited.from_receipt(receipt)
    assert len(events) == 1
    assert events[0].depositor == user.address
    assert events[0].amount == 100
```

#### Property-Based / Fuzz Testing

Instead of testing specific input values, define **properties** that should hold for any valid
input and let the testing framework generate random inputs:

- "A user's balance after deposit should equal their previous balance plus the deposit amount"
- "A user cannot withdraw more than their balance"
- "The total supply should always equal the sum of all balances"

Use Hypothesis (via `ape test` and pytest) or framework-specific fuzzing to generate
random inputs that exercise these properties across a wide range of values.

```python
from hypothesis import given, strategies as st

@given(amount=st.integers(min_value=1, max_value=10**18))
def test_deposit_increases_balance(contract, user, amount):
    initial = contract.balanceOf(user)
    contract.deposit(amount, sender=user, value=amount)
    assert contract.balanceOf(user) == initial + amount
```

#### Invariant Testing

Define invariants — conditions that must **always** be true — and verify them after every
operation:

- Total supply is consistent with individual balances
- Contract's ETH balance matches the sum of all deposits minus withdrawals
- Access control roles are never accidentally granted

#### Gas Usage Tracking

Monitor gas usage to catch unexpected cost increases from code changes:

```python
def test_deposit_gas_usage(contract, user):
    receipt = contract.deposit(100, sender=user, value=100)
    assert receipt.gas_used < 80_000  # Alert if gas usage spikes
```

### Static Analysis

Use **Slither** as a standard part of the development workflow. It catches many common
vulnerability patterns automatically:

```bash
$ slither contracts/
```

Slither is well-supported across the ecosystem and catches issues like:
- Reentrancy vulnerabilities
- Unused state variables
- Missing access controls
- Dangerous delegatecall usage
- Incorrect ERC-20 implementations

## Dependencies

### When to Use External Libraries

Using well-audited implementations of common patterns (like OpenZeppelin for Solidity or
Snekmate for Vyper) is often safer than writing your own. But there is a threshold:

- **Use the dependency** when you need ~95% or more of what it provides. The implementation
  is well-tested and widely audited, and you're using it as intended.
- **Copy and modify** when you need less than ~95% of the functionality. Trying to force-fit
  a dependency that doesn't match your design leads to wrappers, workarounds, and complexity
  that defeats the purpose of using a pre-audited implementation.

### Evaluating Dependencies

Before adopting a contract dependency, check:

- **Audit status** — When was the last audit? What version was audited? Are you using that version?
- **Adoption** — Is it widely used in production? More usage means more eyes and more real-world testing.
- **Maintenance** — Is the project actively maintained? Are security issues addressed promptly?
- **Compatibility** — Does it work with your compiler version and framework?

### Integrating with External Protocols

When your contracts interact with external protocols (DEXs, lending protocols, oracles):

- **Use their interfaces, not their code.** Import the interface (ABI) and interact through it.
  Do not copy their contract code into your project — it makes your codebase harder to audit
  and can create unexpected interactions.
- Treat all external protocol calls as untrusted (see Security > External Call Safety).
- Pin the specific version of the external protocol's interface you are coding against.

## Documentation and Code Style

### NatSpec / Docstrings

- Add NatSpec comments to **all public functions**. These show up in block explorers, developer
  tools, and documentation generators, making the contract more accessible.
- Focus on information that is **not obvious from reading the code**: why a function exists,
  what the caller should know, edge cases, and important preconditions.
- Document `@param` and `@return` values when their purpose isn't self-evident from the name.
- Don't over-describe — if the function is `transfer(to, amount)`, you don't need a paragraph
  explaining that it transfers an amount to an address.

### Naming Conventions

- Keep names **descriptive but not verbose**. A name should be unambiguous in context.
- Follow the conventions of your chosen language:
  - Vyper: `snake_case` for functions and variables, `PascalCase` for contracts/interfaces/events
  - Solidity: `camelCase` for functions and variables, `PascalCase` for contracts/interfaces/events
- Use a consistent naming scheme within the project — consistency matters more than which
  specific convention is chosen.

### Linting and Style

- Use a linter and enforce it in CI. Consistent style across the codebase makes code review
  and auditing significantly easier.
- Slither (https://github.com/crytic/slither) serves double duty as both a static analyzer
  and a style checker for Solidity.
- For Python test code, use `ruff` for formatting and linting.

## Deployment Considerations

### Constructor Design

- Use constructors to set **immutable configuration** that is known at deploy time: token
  addresses, initial parameters, linked contract addresses.
- Keep constructors simple — deploy the contract, set its immutable state, done. Complex
  initialization logic (multi-step setup, external calls) should happen in separate
  transactions after deployment.
- If constructor parameters are complex, document them clearly — the deployer needs to get
  them right on the first try (constructors only run once).

### Multi-Chain Deployments

For mature protocols that deploy across multiple chains:

- Use **deterministic deployment** (CREATE2 / CreateX) so contracts get the same address on
  every chain. This dramatically simplifies cross-chain UX and reduces the amount of
  chain-specific configuration users and integrators need to manage.
- Design contracts so that chain-specific values (e.g. token addresses, protocol endpoints)
  are passed as constructor/initializer arguments rather than hardcoded.
- Reference the `ape/protocol-design` skill for deployment scripting with Ape, including
  CreateX integration for deterministic deployments.

## Key Principles

- **Simplicity is security** — Smaller, simpler contracts are easier to audit and harder to exploit
- **Design by areas of concern** — Group related logic, minimize contract count, avoid complex architectures
- **Immutable by default** — Avoid upgradeability unless there is a specific, justified need
- **Validate at boundaries** — Check inputs at external entry points, trust compiler guarantees internally
- **Readable over clever** — Prefer clear code over gas micro-optimizations; security bugs cost far more than gas
- **Test properties, not just examples** — Use fuzz testing and invariant checks alongside traditional unit tests
- **Evaluate dependencies critically** — Use well-audited libraries when they fit; copy and modify when they don't
- **Emit meaningful events** — Events are the interface between on-chain and off-chain systems; design them carefully
- **Minimize privilege** — Every privileged role is an attack surface; use the least authority needed

## Common Pitfalls

- Don't use inline assembly to save gas — you're giving up compiler safety guarantees for marginal savings
- Don't use complex upgradeability patterns (diamond, custom proxies) without a strong, specific reason
- Don't manually pack storage — the complexity cost outweighs the gas savings until compilers handle it natively
- Don't skip testing revert conditions — knowing that a function fails correctly is as important as knowing it succeeds
- Don't copy external protocol code into your project — import their interfaces instead
- Don't add roles and admin functions "just in case" — every privileged function is an attack surface
- Don't optimize view functions aggressively — they're free when called off-chain
- Don't use unbounded loops over storage arrays — they will eventually hit the gas limit
- Don't trust that external calls will behave as expected — always validate return values and handle failures
- Don't change compiler versions between audit and deployment — this invalidates the audit
