---
name: writing-sdks
description: |
  Create a Python SDK using Ape Framework to simplify using a given blockchain-based protocol.
  Use when users need specific best practices and guidance on developing a new integration with the given protocol.
  Do not use if such a Python SDK already exists that is designed to work with Ape.
compatibility: Requires uv installed
---

# Overview

Python library for interacting with a specific protocol (e.g., Uniswap, Aave, custom protocol).

## Prerequisites

Before using this skill, verify the user has:

- `uv` installed (https://docs.astral.sh/uv)
- A link to Protocol's official documentation
- A link to Protocol's Github organization

### Folder Structure

```
[protocol-name]/
├── pyproject.toml          # Contains protocol version(s) as dependencies in `[tool.ape.dependencies]`
├── README.md               # Overview, Quick Usage guide, any Config or Environment Variables used
├── src/
│   └── [protocol_name]/
│       ├── __init__.py     # Module exports (e.g. `from .main import ProtocolName; __all__ = ["ProtocolName", ...]`)
│       ├── __main__.py     # Protocol use CLI entrypoint (if requested by user)
│       ├── main.py         # Main SDK class (e.g. `from protocol_name import ProtocolName`)
│       ├── types.py        # Special type definitions and data models/ABCs for all versions
│       ├── v*.py           # Version-specific implementations (implementing models/ABCs from `types.py`)
│       ├── packages.py     # Code for loading types (by version) from `manifests/`
│       ├── manifests/      # JSON artifacts bundled with SDK (load via `ethpm_types.PackageManafest.model_validate_json`)
│          └── v*.json      # Package manifests (updated via `ape run build` script)
│       ├── utils.py        # Utility functions
│       └── exceptions.py   # Custom exceptions
├── tests/
│   ├── conftest.py         # Pytest fixtures
│   └── functional/         # Functional test cases (might require fork testing)
│       ├── test_main.py    # Test main SDK class
│       ├── test_v*.py      # Test more complex cases for a specific version
│       └── ...             # Other tests (Custom exceptions, functions from utils, etc.)
├── scripts/
│   └── build.py            # Build script (copy files from `~/.ape/packages/` to `src/[protocol_name]/manifests/`)
└── contracts/              # If managing contract ABIs locally (instead of with `ape pm install && ape run build`)
    └── abis/
        └── Type.json       # Contract type (updated via `ape run build` script)
```

### Key Files

**`src/[protocol_name]/main.py`**

- Main entry point for SDK users (e.g. `from protocol_name import ProtocolName; name = ProtocolName()`)
- Manages version-specific loading (e.g. `ProtocolName(use_v2=False)`
- Provides high-level API methods with Ape-enhanced method call kwargs (e.g. `name.action(..., sender=me)`)

**`src/[protocol_name]/packages.py`**

- Wrapper classes around protocol smart contract package(s)
- Contains import system-level contract address(es) for all supported chain deployment(s) and types (used in SDK)

**`src/[protocol_name]/types.py`**

- Pydantic models and/or Abstract Base Classes (ABCs)
- Genericized to work for all supported protocol versions
- Enums for protocol states/options (using `str` or `int` variants for integration w/ contract calls)

### Dependencies (partial)

```toml
dependencies = [
    "eth-ape>=0.8.0",
    ...  # Other specially-required packages for SDKs
]
```

### Example Structure

```python
# packages.py
from importlib import resources

from ape.managers.project import ProjectManager

if TYPE_CHECKING:
    from ape.contracts import ContractContainer
    from ape.types import AddressType

root = resources.files(__package__)


class PackageType(str, Enum):
    # NOTE: Common types used across versions
    FACTORY = "Factory"
    POOL = "Pool"
    ...

    def __call__(self, version: str) -> "ContractContainer":
        with resources.as_file(root.joinpath(f"manifests/{version}.json")) as manifest_json_file:
            if not manifest_json_file.exists():
                raise Exception(...)

            VersionTypes = ProjectManager.from_manifest(manifest_json_file)

        match self:
            case PackageType.FACTORY:
                return VersionTypes.ProtocolV2Factory

            case PackageType.POOL:
                return ...

            case ...:
                return ...


...  # NOTE: Load manifests for other protocol versions


DEPLOYMENTS_BY_CHAIN_ID: dict[str, dict[int, "AddressType"] = {
    "ProtocolV2Factory": {
        0: "0xAbCd...1234",  # There is no chain ID 0, therefore this is the "generic" instance address
        1: ...,  # Chain-specific instance (if different from generic)
        ...  # NOTE: All supported chain IDs for type that have specific deployments
    },
    ...,  # NOTE: Other important types
}


def get_address(ContractType: "ContractContainer", chain_id: int) -> "AddressType":
    if not (instances := DEPLOYMENTS_BY_CHAIN_ID.get(ContractType.name):
        raise Exception(...)

    return instances.get(chain_id, instances[0])  # Return specific instance, fallback to "generic"
```

```python
# v2.py
from typing import TYPE_CHECKING

from ape.utils import ManagerAccessMixin

from .packages import PackageType, get_address

if TYPE_CHECKING:
    from ape.api import ReceiptAPI
    from ape.contracts import ContractInstance
    from ape.types import AddressType


class Factory(ManagerAccessMixin):
    def __init__(self, address: "AddressType | None" = None):
        if address:
            # NOTE: Override with custom address
            self.address = address

    @cached_property
    def address(self) -> "AddressType":
        return get_address(
            PackageType.FACTORY("v2"),
            chain_id=self.chain_manager.chain_id,
        )

    @cached_property
    def contract(self) -> "ContractInstance":
        return self.chain_manager.contracts.instance_at(
            self.address,
            contract_type=PackageType.FACTORY("v2"),
        )

    def action(self, ..., **txn_args) -> "ReceiptAPI":
        ...  # Input validation (raise custom errors from `exceptions.py`)
        return self.contract.action(..., **txn_args)
```

```python
# main.py
from . import v2, v3, ...


class ProtocolName:
    def __init__(self, use_v2: bool=True, ...):
        self.v2 = v2.Factory() if use_v2 else None
        ...

    def action(self, ..., **txn_args) -> "ReceiptAPI":
        ...

        if self.v2:
            return self.v2.action(..., **txn_args)

        else:
            ...
```
