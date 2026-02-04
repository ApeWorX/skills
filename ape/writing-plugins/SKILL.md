---
name: writing-plugins
description: |
  Create a new Plugin for Ape Framework to integrate a specific tool or service into one of Ape's vertical(s).
  Use when users need specific best practices and guidance on developing a new plugin for Ape.
  Do not use if such a plugin already exists that is designed to work with Ape, and instead suggest that.
compatibility: Requires uv installed
---

# Overview

Create a Plugin for Ape Framework that adds new functionality to the framework.

### Plugin Types

An Ape plugin can customize support in 1 or more Ape vertical(s).
Those Verticals are:

- **Compiler Plugins** - Support new smart contract languages
- **Provider Plugins** - Connect to ecosystem network(s) using a specialized service
- **Query Plugins** - Source important data queries using a specialized service or tool
- **Conversion Plugins** - Support specialized conversions of Python types into ABI supported types

Additionally, if required an Ape plugin can also define some extensions to core Ape functionality:

- **Config Extensions** - Custom configuration subfields specifically for the plugin
- **CLI Extensions** - Add new subcommands to `ape` CLI, e.g. `ape plugin-name ...`

These extensions are usually added to help otherwise manage one of the plugin vertical(s) mentioned above.

## Prerequisites

Before using this skill, verify the user has:

- `uv` installed (https://docs.astral.sh/uv)
- A link to the Tool or Service's official documentation
- A link to Tool or Service's Github repository

## Folder Structure

When creating a new plugin, use the following modified folder structure to organize the plugin in a consistent way:

```
ape-[plugin-name]/                  # Project and package name should be `ape-[plugin-name]`
├── ape_[plugin_name]/              # Main project Python module
│   ├── __init__.py                 # Must contain Ape plugin registration (e.g. `plugins.register(plugins.[VerticalType])`)
│   ├── __main__.py                 # Ape CLI subcommand (if applicable), top-level `click.Group` must be `cli`
│   ├── [vertical_type]s.py         # Implementation for each vertical supported in plugin (e.g. `compilers.py`, `providers.py`, etc.)
│   ├── client.py                   # Service client (if plugin is for a service)
│   └── config.py                   # Plugin configuration (if applicable)
└── tests/                          # Tests run with `ape test`
    ├── ...                         # Other test files (`conftest.py`, `integration/`, etc.)
    └── functional/                 # Add tests for plugin implementations (may use `pytest-mock` for services)
        ├── test_[vertical_type].py # Test(s) for each plugin vertical implementation
        └── ...                     # Other unit tests (`client.py`, `utils.py`, etc.)
```

**Important Notes**:

1. **`pyproject.toml`**

- Make sure to add a pin the plugin to the latest minor version of Ape:

```toml
[project]
...
dependencies = [
    "eth-ape>=0.8,<0.9",
    ...  # Additional dependencies here
]
```

- If adding a CLI extension to Ape (e.g. `ape [plugin-name]`), must declare an entrypoint as follows:

```toml
[project.entry-points."ape_cli_subcommands"]
[plugin_name] = "ape_[plugin_name].__main__:cli"
```

2. **`ape_[plugin_name]/__init__.py`**

- Ape Plugins require an additional registration step in order to function with Ape:

```python
from ape.plugins import register, PluginTypeA, ...

@register(PluginTypeA)
def plugin_types():
    # NOTE: Support lazy-loading plugin functionality only as needed
    from .plugin_type import YourPluginClass

    return [YourPluginClass]

# NOTE: Do not have any default exports unless required for other purposes
```
