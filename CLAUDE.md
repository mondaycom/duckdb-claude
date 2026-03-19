# DuckDB Development Guide

## Repository Structure

- `src/` — Core C++ source code (catalog, common, execution, function, main, optimizer, parallel, parser, planner, storage, transaction)
- `src/include/` — Header files
- `extension/` — In-tree extensions (parquet, json, icu, tpch, tpcds, autocomplete, core_functions, jemalloc, delta)
- `test/` — Tests (sqllogictest `.test` files and C++ unit tests)
- `test/sql/` — SQL-based tests organized by category (aggregate, join, cast, function, etc.)
- `third_party/` — Third-party dependencies
- `tools/` — Language bindings and tools
- `scripts/` — Build and CI scripts
- `.github/config/` — CI extension build configurations

## Building

DuckDB uses CMake + Make. Always run from the repo root.

```bash
# Release build (default)
make

# Debug build (includes sanitizers and DEBUG_MOVE)
make debug

# RelWithDebInfo + assertions (used in CI smoke tests)
make relassert

# Use Ninja for faster parallel builds
GEN=ninja make

# Limit parallel build jobs if system runs out of memory
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make

# Build with specific extensions
BUILD_EXTENSIONS="httpfs;json;icu" make

# Build all in-tree extensions (same as CI)
EXTENSION_CONFIGS=.github/config/in_tree_extensions.cmake make

# Build all extensions including out-of-tree (requires vcpkg)
BUILD_ALL_EXT=1 make
```

Build outputs go to `build/<type>/` (e.g., `build/debug/`, `build/release/`).

The build system automatically detects and uses `ccache` or `sccache` if installed — use them to speed up rebuilds.

## In-Tree Extensions (Built in CI)

The CI pipeline builds these in-tree extensions (`.github/config/in_tree_extensions.cmake`):
- `autocomplete`, `core_functions`, `icu`, `json`, `parquet`, `tpcds`, `tpch`

The base config (`extension/extension_config.cmake`) always includes: `core_functions`, `parquet` (and `jemalloc` on 64-bit Linux).

For local extension customization, create `extension/extension_config_local.cmake` (git-ignored).

## Running Tests

**IMPORTANT: Never run all tests at once. Always run specific tests or test groups.**

### Running a specific test file

```bash
# Debug build
build/debug/test/unittest "test/sql/aggregate/aggregates/my_test.test"

# Release build
build/release/test/unittest "test/sql/aggregate/aggregates/my_test.test"
```

### Running tests by group/tag

```bash
# Run tests matching a tag
build/debug/test/unittest "[sql]"

# Run tests matching a pattern
build/debug/test/unittest "*test_name*"
```

### Running tests with specific configs

```bash
# With a test config (e.g., verification enabled)
build/debug/test/unittest "test/path/to/test.test" --test-config="test/configs/enable_verification.json"
```

Available test configs are in `test/configs/` (e.g., `enable_verification.json`, `force_storage.json`, `latest_storage.json`, `verify_fetch_row.json`).

### CI-style test runner (parallel, with retries)

```bash
# Run all tests via the CI runner (parallel, with batching)
python3 scripts/ci/run_tests.py build/relassert/test/unittest

# Run smoke tests
make smoke

# Run with a specific test filter
python3 scripts/ci/run_tests.py build/relassert/test/unittest "pattern"
```

### Test file types

- `.test` — Standard sqllogictest files (run in fast test suite)
- `.test_slow` — Slower tests (run only in `allunit` / nightly CI)

**Strongly prefer writing tests as sqllogictest `.test` files** rather than C++ tests. Only use C++ for tests requiring concurrent connections or exotic behavior.

## Formatting

```bash
# Check formatting
make format-check

# Fix formatting (run before submitting PRs)
make format-fix

# Format only changed files (relative to main)
make format-main
```

Uses `clang-format` 11.0.1 and `black`. Install with: `python3 -m pip install clang-format==11.0.1`

Rules: tabs for indentation, spaces for alignment, 120 column max line length.

## Code Generation

Some source files are auto-generated. After modifying schemas or adding settings/functions:

```bash
make generate-files
```

This runs several Python scripts to regenerate C API bindings, function registrations, settings, serialization code, etc.

## C++ Conventions

- Use `duckdb` namespace for all code in `src/`
- Use `unique_ptr` over `shared_ptr`; no raw `new`/`delete`/`malloc`
- Use `[u]int(8|16|32|64)_t` for sized integers, `idx_t` for indices/counts
- CamelCase for types and functions, snake_case for variables and files
- Use `D_ASSERT` for programmer-error assertions (never triggered by user input)
- Use exceptions only for query-terminating errors
- `override` or `final` on virtual method overrides; never repeat `virtual`
- Do not `using namespace std`

## Debugging Tips

- Debug builds include AddressSanitizer and UBSan by default. Disable with `DISABLE_SANITIZER=1 make debug`
- Use `FORCE_ASSERT=1` in release builds to enable assertions without full debug overhead
- The `relassert` build type is RelWithDebInfo + assertions — good balance of speed and safety
- Use `--test-temp-dir <path>` with unittest to preserve test artifacts for inspection

## Key CI Workflows

- PR checks: format check, debug build + fast unit tests, release build + all unit tests
- Nightly: extended tests, package builds, storage compatibility checks
- Extension builds use configs from `.github/config/`

## Warnings

- **Do NOT run the DuckDB CLI with queries that may run forever** (e.g., complex aggregation queries). Use the unittest binary for testing.
- **Do NOT run `make allunit`** — it takes ~1 hour. Run specific tests instead.

## Git Commits

- **Never add Claude as a co-author** in commit messages. Do not include `Co-authored-by: Claude` or any similar line attributing Claude as a contributor.
