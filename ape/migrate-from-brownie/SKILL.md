---
name: migrate-from-brownie
description: |
  Migrate a Python Ethereum smart-contract project from the deprecated Brownie framework to ApeWorx Ape.
  Use when users have an existing Brownie project (`brownie-config.yaml` present, `from brownie import …` in their .py files) and want to upgrade to the actively-maintained Ape Framework.
  Guides the user through running the community codemod, applying AI-assisted manual cleanup for patterns the codemod intentionally leaves as TODOs, and verifying the migration with `ape compile` + `ape test`.
compatibility: Requires Node.js 20+ (for `npx codemod`) and Python 3.10+ (for `eth-ape`)
---

# Overview

Brownie was the dominant Python framework for Ethereum smart-contract development from 2019, [officially deprecated in 2023](https://github.com/eth-brownie/brownie/blob/master/README.md) in favor of Ape Framework. Many existing projects still depend on Brownie. This skill guides users through migrating a real Brownie project to Ape using:

1. **Deterministic codemod** ([`@pugarhuda/brownie-to-ape`](https://github.com/PugarHuda/brownie-to-ape)) for the ~85% of patterns that are mechanical AST rewrites.
2. **AI-assisted manual cleanup** for the remaining ~15% — patterns flagged with `# TODO(brownie-to-ape):` comments that require project schema introspection (contract artifact references, etc.) the codemod can't safely auto-resolve.
3. **End-to-end verification** with `ape compile` + `ape test`.

The codemod is validated on 5 OSS Brownie projects including [`yearn/brownie-strategy-mix`](https://github.com/yearn/brownie-strategy-mix) with **zero false positives** across the entire validation set. End-to-end on `brownie-mix/token-mix`: `ape test --network ::test` → **38 passed, 0 failed in 5.40s**.

## Prerequisites

Before using this skill, verify the user has:

- A Brownie project at a known path (look for `brownie-config.yaml` and `from brownie import …` in `.py` files)
- Node.js 20+ available (`node --version`)
- Python 3.10+ available (`python --version`)
- Their working tree clean or backed up

If any of these are missing, help the user install them before continuing.

## Workflow

### Step 1: Apply the codemod

```bash
npx codemod@latest @pugarhuda/brownie-to-ape -t /absolute/path/to/their/brownie/project
```

Expected behavior:

- Completes in ~3 seconds (after `npx` warmup on first run)
- Modifies between 4 and 22 `.py` files depending on project size
- Inserts `# TODO(brownie-to-ape):` comments where the codemod intentionally cannot auto-resolve
- Never touches `.sol` files or non-Brownie Python files

### Step 2: Migrate the YAML config

```bash
git clone --depth 1 https://github.com/PugarHuda/brownie-to-ape /tmp/brownie-to-ape
python /tmp/brownie-to-ape/scripts/migrate_config.py /absolute/path/to/their/project
```

Translates network configs, dependencies, solc remappings, and dotenv settings. Unrecognized fields produce a TODO list at the top of `ape-config.yaml` for manual review.

### Step 3: Inspect the codemod's TODO comments

```bash
grep -rn "TODO(brownie-to-ape)" --include="*.py" /absolute/path/to/their/project
```

Typical output: 1–6 TODOs per repo, in three common categories:

| TODO pattern | What to do |
|---|---|
| `# TODO(brownie-to-ape): no direct Ape equivalent for: <Name>` | Contract artifact deploy. Add `from ape import project` and prefix with `project.<Name>.deploy(...)`. |
| `# TODO(brownie-to-ape): Ape has built-in per-test chain isolation via chain.isolate(). This fixture can be removed.` | Brownie's `def isolate(fn_isolation): pass` fixture is unnecessary in Ape. Delete the fixture entirely. |
| `# TODO(brownie-to-ape):` for `accounts.add(pk)`, `interface.IERC20(addr)`, `Web3.toWei(...)` | Apply Ape API: `accounts.import_account_from_private_key(...)`, `Contract(addr)`, or `convert(...)`. |

### Step 4: Apply AI-assisted manual cleanup

**Contract artifact prefix:**

```diff
+from ape import project

 @pytest.fixture(scope="module")
-def token(Token, accounts):
-    return Token.deploy("Test", 18, 1e21, sender=accounts[0])
+def token(accounts):
+    return project.Token.deploy("Test", 18, int(1e21), sender=accounts[0])
```

Note `int(1e21)` — Ape rejects float scientific notation with `ConversionError`.

**Drop the isolate fixture:**

```diff
-@pytest.fixture(scope="function", autouse=True)
-def isolate(fn_isolation):
-    pass
```

**Test files lacking `import brownie`** (codemod's FP-guard skipped them — but they may still use `{'from': X}`):

```diff
-token.approve(accounts[1], 10**19, {'from': accounts[0]})
+token.approve(accounts[1], 10**19, sender=accounts[0])
```

**Event log access semantics:**

```diff
-assert tx.events["Transfer"].values() == [accounts[0], accounts[1], amount]
+logs = list(tx.decode_logs(token.Transfer))
+assert len(logs) == 1
+assert getattr(logs[0], "from") == accounts[0]   # `from` is a Python keyword
+assert logs[0].to == accounts[1]
+assert logs[0].value == amount
```

**Transaction return value:**

```diff
-assert tx.return_value is True
+assert token.allowance(accounts[0], accounts[1]) == 10**19
```

### Step 5: Install Ape + plugins, then verify

```bash
pip install eth-ape
ape plugins install solidity --yes
cd /absolute/path/to/their/project
ape compile
ape test --network ::test
```

Reference run on `brownie-mix/token-mix` after codemod + ~30 LOC of AI-step fixes:

```
$ ape compile
SUCCESS: 'local project' compiled. (solc 0.6.12, 2 contracts)

$ ape test --network ::test
======================== 38 passed, 0 failed in 5.40s ========================
```

Full passing log: https://github.com/PugarHuda/brownie-to-ape/blob/main/docs/ape-verify-token-mix.log
Step-by-step walkthrough: https://github.com/PugarHuda/brownie-to-ape/blob/main/demo/ai-step-demo.md

## Common gotchas

1. **`ConversionError: No conversion registered to handle '1e+21'`** — Ape rejects Python float scientific notation. Wrap with `int(...)`.
2. **`fn_isolation` fixture not found** — Brownie-specific; delete the `isolate(fn_isolation)` fixture entirely.
3. **`'ContractLog' object has no attribute '_from'`** — Ape exposes the field as `from` (Python keyword); use `getattr(log, "from")`.
4. **`ArgumentsLengthError`** — typically a `{'from': X}` dict still present in a file the codemod skipped (no `import brownie` line). Convert to `sender=X` manually.
5. **`Token.deploy` returns `None`** — missing `project.` prefix.

## What the codemod auto-handles (deterministic, 0 FP)

17 transform passes covering: imports rewrite (`from brownie import …` → `from ape import …` with `network` → `networks`), constants relocation (`ZERO_ADDRESS` → `ape.utils`), `brownie.<attr>` → `ape.<attr>`, tx-dict to kwargs, `network.show_active()` rewrite, `chain.mine(N)` / `chain.sleep(N)` API changes, `Wei("X")` → `convert("X", int)` with auto-import, exception class renames, whale impersonation idiom (`accounts.at(addr, force=True)` → `accounts.impersonate_account(addr)`), and more.

## What the codemod intentionally does NOT handle (left as TODOs)

- **Contract artifacts** — requires project schema introspection unavailable to the jssg sandbox.
- **`accounts.add(private_key)`** — Ape's equivalent requires alias + passphrase that don't exist in the Brownie call.
- **`interface.IERC20(addr)`** — Ape's `Contract(addr)` requires an ABI registered for the interface name.
- **Custom test fixtures with non-trivial bodies** — codemod only touches the trivial `def isolate(fn_isolation): pass` form.

By design: the codemod prefers FN over FP, so when in doubt it inserts a TODO and stops.

## References

- **Codemod registry:** https://app.codemod.com/registry/@pugarhuda/brownie-to-ape
- **Source code:** https://github.com/PugarHuda/brownie-to-ape
- **Official Ape Brownie migration guide:** https://docs.apeworx.io/ape/latest/userguides/brownie-migration.html
- **End-to-end passing log:** https://github.com/PugarHuda/brownie-to-ape/blob/main/docs/ape-verify-token-mix.log
- **AI-step demo (token-mix):** https://github.com/PugarHuda/brownie-to-ape/blob/main/demo/ai-step-demo.md
- **Engineering tradeoffs:** https://github.com/PugarHuda/brownie-to-ape/blob/main/docs/DEFERRED_FEATURES.md
