# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

mdformat is a CommonMark compliant Markdown formatter - an opinionated tool that enforces a consistent style in Markdown files. It works as both a command-line tool and a Python library, with a plugin-based architecture for extensibility.

## Development Commands

### Testing
```bash
# Run all tests across Python 3.10-3.13
tox

# Run tests for specific Python version
tox -e 3.11

# Run tests with coverage (quick)
pytest --cov

# Run single test file
pytest tests/test_cli.py

# Run specific test
pytest tests/test_cli.py::test_name
```

### Linting and Type Checking
```bash
# Run all linters (uses pre-commit hooks)
tox -e pre-commit

# Run specific linters
pre-commit run -a

# Type checking with mypy
tox -e mypy
# or directly:
mypy src/ tests/

# Format code with mdformat itself
mdformat --check docs/ README.md
```

### Pre-commit Hook Testing
```bash
# Test mdformat's own pre-commit hook
tox -e hook
# or:
pre-commit try-repo . mdformat --files README.md
```

### Other Commands
```bash
# Build documentation
tox -e docs

# Run profiler
tox -e profile

# Run fuzzer
tox -e fuzz

# Benchmark mdformat
tox -e benchmark

# CLI testing
tox -e cli -- [args]
```

## Architecture

### Core Flow: Parsing → Rendering → Validation

1. **Parser** (`markdown-it-py`): Converts Markdown text into tokens
2. **Renderer** (`MDRenderer`): Converts tokens back to formatted Markdown
3. **Validation** (optional): Verifies HTML rendering consistency

### Key Components

#### API Layer (`src/mdformat/_api.py`)
- `text(md, options, extensions, codeformatters)`: Format Markdown string
- `file(path, options, extensions, codeformatters)`: Format file in place
- Two-pass rendering: When word wrap changes, renders twice to stabilize escapes

#### CLI Layer (`src/mdformat/_cli.py`)
- File path resolution (supports globs, directories, stdin)
- Config file discovery and merging (`.mdformat.toml`)
- Options precedence: DEFAULT_OPTS < TOML config < CLI args
- Plugin option merging under `opts["plugin"][<plugin_id>]`

#### Plugin System (`src/mdformat/plugins.py`)
Two plugin types discovered via entry points:

1. **Parser Extensions** (`mdformat.parser_extension`):
   - `update_mdit(mdit)`: Configure markdown-it parser
   - `RENDERERS`: Map syntax types to render functions
   - `POSTPROCESSORS`: Post-process rendered output (collaborative)
   - `CHANGES_AST`: Flag if plugin modifies Markdown AST
   - Optional CLI arguments via `add_cli_argument_group()`

2. **Code Formatters** (`mdformat.codeformatter`):
   - Function signature: `(code: str, info: str) -> str`
   - Entry point name = language name in code block info string

#### Renderer (`src/mdformat/renderer/`)
- `MDRenderer`: Main renderer class (markdown-it-py compatible)
- `RenderTreeNode` (`_tree.py`): Tree structure for token hierarchy
- `RenderContext` (`_context.py`): Shared rendering state
- `DEFAULT_RENDERERS`: Built-in render functions per syntax type
- Renderer functions merged: defaults + plugin overrides
- Postprocessors run in series (all plugins contribute)

### Configuration Files

- `.mdformat.toml`: Project-specific options
- Config searched upward from file being formatted
- Options: wrap, end_of_line, number, exclude (Python 3.13+), extensions, codeformatters
- Plugin-specific options under `[plugin.<plugin_id>]` section

## Testing Notes

- Test fixtures use `FIXTURE_MD` format (input/output pairs)
- CommonMark spec compliance tested in `test_commonmark_spec.py`
- Tests require: pytest, pytest-randomly, pytest-cov, covdefaults
- Coverage tracked via covdefaults plugin

## Plugin Development

See `docs/contributors/contributing.md` for detailed plugin development guide.

**Code Formatter Plugin Entry Point:**
```toml
[project.entry-points."mdformat.codeformatter"]
"python" = "my_package.module:format_python"
```

**Parser Extension Plugin Entry Point:**
```toml
[project.entry-points."mdformat.parser_extension"]
"my_extension" = "my_package.module"
```

## Important Constraints

- Python 3.10+ required
- CommonMark strict compliance - custom syntax needs plugins
- Opinionated tool - minimal configuration philosophy
- Two-pass rendering needed when word wrap changes
- HTML validation ensures formatting doesn't alter rendered output
