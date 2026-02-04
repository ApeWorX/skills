---
name: python-packaging
description: |
  Best practices for developing a Python-based project.
  Use when creating a new project to ensure best practices according to ApeWorX development standards.
compatibility: Requires uv installed
---

# Overview

This skill specifies best practices for organization and tooling when developing Python projects.

## Prerequisites

Before using this skill, verify the user has:

- `uv` installed (https://docs.astral.sh/uv)
- A name for the project (e.g. `project-name`)
- An unused PyPI package name for the project (e.g. `package-name`), if developing as a new project

## Folder Structure

When creating a new project, use the following folder structure to organize the project in a consistent way:

```
[project-name]/                     # Root of project
├── pyproject.toml                  # Must include ape plugin entry point (if subcommand defined)
├── README.md                       # Top-level information about the project, must include Usage Guide
├── [package_name]/                 # Main project Python module
│   ├── __init__.py                 # Must contain top-level exports for package in `__all__`
│   ├── __main__.py                 # CLI command for package (if applicable), top-level `click.Group` must be `cli`
│   └── ...                         # Other package files (`types.py`, etc.)
├── tests/                          # Tests run with `ape test`
│   ├── conftest.py                 # Fixtures needed to run tests
│   ├── functional/                 # Organize functional/unit tests here
│   │   └── test_[component].py     # Test a specific functional component from the package
│   └── integration/                # Organize integration tests here (if there is a CLI or complex integration)
│       └── test_cli.py             # Test a specific functional component from the package
└── docs/                           # Project documentation, if requested (managed with `sphinx-ape`)
    ├── methoddocs/                 # Autodocs for package (add important classes, functions, etc.)
    ├── commands/                   # Autodocs for cli commands (if any)
    ├── userguides/                 # Any specific userguide(s) needed to explain how to use plugin
    └── conf.py                     # `sphinx-ape` config
```

### Key Files

#### **`pyproject.toml`**

This file is used for maintaining the project's Python package information,
use the [official packaging userguide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml)
for help setting up this file properly.

**Important Notes**

1. We prefer to use `setuptools-scm` to dynamically manage the version of the project using git tags.
   This makes releasing the package very easy:

```toml
[build-system]
requires = ["setuptools>=75.6", "wheel", "setuptools_scm[toml]>=5"]
build-backend = "setuptools.build_meta"
...

[project]
name = "[package-name]"
dynamic = [ "version" ]
...

[tool.setuptools_scm]
# NOTE: This is required for `setuptools_scm` to work
```

2. We prefer to use `dependency-groups` to manage dev-only dependencies:

```toml
...

[dependency-groups]
test = [  # `test` GitHub Action jobs uses this
    "pytest>=9.0,<10",  # Preferred testing framework
    "pytest-cov>=4.0,<5",  # Coverage analysis for package
    ...  # NOTE: Any other test-only dependencies
]
lint = [
    "ruff>=0.14",  # Unified linter and formatter
    "mypy>=1.16,<2",  # Static type analyzer
    ...  # NOTE: Add any required `types-*` packages required here
    "mdformat>=0.7",  # Auto-formatter for markdown
    "mdformat-gfm>=0.3.5",  # Needed for formatting GitHub-flavored markdown
    "mdformat-frontmatter>=0.4.1",  # Needed for frontmatters-style headers in issue templates
    "mdformat-pyproject>=0.0.2",  # Allows configuring in pyproject.toml
]
docs = [
    "sphinx-ape>=0.1,<1",  # Our preferred documentation plugin
]
dev = [
    "commitlint",  # Check for adherence to conventionalcommits
    "pre-commit",  # Ensure that linters are run prior to committing
    "pytest-watch",  # Automatic test watcher/runner
    "pytest-xdist",  # Multi-process test runner
    "ipdb",  # Debugger (Must activate with `export PYTHONBREAKPOINT=ipdb.set_trace`)
    {include-group = "test"},
    {include-group = "lint"},
    {include-group = "docs"},
]
```

#### **`README.md`**

This file is used on Github to display top-level information about project, how it should be used, etc.
It is important that this file have relevant information for downstream users in order to use it correctly.

**Important Notes**

1. It should contain an "Installation" section, which must describe dependencies required to use it.
   (Is this an Ape plugin or project requiring `eth-ape` installed? A Silverback bot requiring `silverback` installed?)
2. It should contain a "Usage Guide" section, which must describe the basic usage of the top-level exports from the package.
3. If our documentation plugin is configured for the project, this will become the "Quickstart Userguide" for the project.

#### **`tests/`**

Test organization is important for the project,
we typically segregate our test suite into "functional" (unit-level) tests and "integration" (system-level) tests.
Doing this makes it easy to run only the relevant tests when developing a new feature.
