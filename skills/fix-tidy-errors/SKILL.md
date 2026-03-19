---
name: fix-tidy-errors
description: Fix clang-tidy errors that would fail the tidy-check CI stage
disable-model-invocation: true
---

# Fix Tidy-Check CI Errors

Identify and fix all clang-tidy errors in the files changed on this branch that would fail the `tidy-check` CI stage.

## Step 1: Locate clang-tidy

Check whether `clang-tidy` is available:
```bash
which clang-tidy || echo "not found"
```

- **macOS**: Homebrew LLVM is keg-only, so `clang-tidy` is not on PATH by default even after `brew install llvm`. Locate it with:
  ```bash
  find $(brew --prefix llvm)/bin -name 'clang-tidy' 2>/dev/null
  ```
  If not installed: `brew install llvm`. The binary will be at `$(brew --prefix llvm)/bin/clang-tidy`.
  Store this path — you'll need it as `TIDY_BINARY` in subsequent steps.

- **Ubuntu/Debian**: `sudo apt-get install -y clang-tidy`. It will be on PATH automatically.

## Step 2: Verify branch

Confirm we are not on `main`:
```bash
git rev-parse --abbrev-ref HEAD
```
Stop if the result is `main`.

## Step 3: Ensure the tidy build exists

The tidy build must be configured before running the diff check:
```bash
mkdir -p ./build/tidy && cd build/tidy && cmake -DCLANG_TIDY=1 -DDISABLE_UNITY=1 -DBUILD_EXTENSIONS=parquet -DBUILD_SHELL=0 ../..
```
This only runs cmake configuration (fast). Skip if `build/tidy/compile_commands.json` already exists.

## Step 4: Run tidy-check-diff and capture errors

Run the diff-based tidy check against `main`. The make variable is `GIT_BASE_BRANCH` (not `DUCKDB_GIT_BASE_BRANCH`):

**Linux** (clang-tidy on PATH):
```bash
GIT_BASE_BRANCH=main make tidy-check-diff 2>&1
```

**macOS** (clang-tidy is keg-only; system headers require explicit sysroot):

`make tidy-check-diff` cannot pass the sysroot flag, so invoke the diff script directly:
```bash
TIDY_BINARY=$(find $(brew --prefix llvm)/bin -name 'clang-tidy') && \
SDK=$(xcrun --show-sdk-path) && \
git diff origin/main . ':(exclude)tools' ':(exclude)extension' ':(exclude)test' ':(exclude)benchmark' ':(exclude)third_party' ':(exclude)src/common/adbc' ':(exclude)src/main/capi' | \
python3 scripts/clang-tidy-diff.py \
  -path build/tidy \
  -quiet \
  -clang-tidy-binary "$TIDY_BINARY" \
  -extra-arg="-isysroot${SDK}" \
  -p1 2>&1
```

> **macOS note**: Without `-extra-arg="-isysroot..."`, Homebrew LLVM clang-tidy cannot find system headers (`<memory>`, `<sstream>`, etc.) and emits `clang-diagnostic-error: file not found` for every file. These are **not real code errors** — they are a local toolchain issue. Ignore them if the only errors are system header `file not found` diagnostics; they will not occur in Linux CI.

Capture the full output. If there are no errors (or only the macOS system-header false positives described above), report success and stop.

## Step 5: Parse and group errors

From the tidy output, group errors by file and by check name. Each diagnostic line looks like:
```
src/some/file.cpp:42:10: error: [check-name] message
```

Common checks and how to fix them:

| Check | Fix |
|---|---|
| `modernize-use-nullptr` | Replace `NULL` or `0` used as pointer with `nullptr` |
| `modernize-use-override` | Add `override` to virtual method overrides; remove redundant `virtual` keyword |
| `google-explicit-constructor` | Add `explicit` to single-argument constructors |
| `google-build-using-namespace` | Remove `using namespace std;` (or other namespaces); qualify names instead |
| `google-runtime-int` | Replace `short`/`long`/`unsigned long` etc. with sized types (`int16_t`, `int64_t`, `uint64_t`, etc.) |
| `readability-braces-around-statements` | Add braces `{}` around the body of `if`/`else`/`for`/`while` even for single-statement bodies |
| `readability-container-size-empty` | Replace `.size() == 0` / `.size() != 0` with `.empty()` / `!.empty()` |
| `modernize-use-bool-literals` | Replace integer literals `0`/`1` used as booleans with `false`/`true` |
| `modernize-use-emplace` | Replace `.push_back(T(...))` with `.emplace_back(...)` for smart pointers listed in config |
| `cppcoreguidelines-pro-type-cstyle-cast` | Replace C-style casts `(Type)x` with `static_cast<Type>(x)`, `reinterpret_cast`, or `duckdb_py_cast` as appropriate |
| `cppcoreguidelines-pro-type-const-cast` | Avoid `const_cast`; redesign to remove the need |
| `cppcoreguidelines-rvalue-reference-param-not-moved` | Call `std::move()` on rvalue reference parameters that are passed to functions |
| `cppcoreguidelines-virtual-class-destructor` | Add a `virtual` destructor to any class with virtual methods |
| `cppcoreguidelines-slicing` | Pass polymorphic objects by pointer or reference, not by value |
| `hicpp-exception-baseclass` | Ensure thrown types inherit from `std::exception` |
| `misc-throw-by-value-catch-by-reference` | Throw by value, catch by `const` reference |
| `performance-*` | Fix as described in the diagnostic message |
| `bugprone-*` | Fix as described in the diagnostic message |
| `readability-identifier-naming` | Rename to match convention: `CamelCase` for classes/functions/enums, `lower_case` for variables/members/parameters, `UPPER_CASE` for static constants/enum values/macros, `_t` suffix for typedefs |

**DuckDB-specific conventions** (from `CLAUDE.md`):
- Use `unique_ptr` over `shared_ptr`; no raw `new`/`delete`
- Use `idx_t` for indices/counts, `[u]int(8|16|32|64)_t` for sized integers
- Use `D_ASSERT` for programmer-error assertions
- `override` or `final` on virtual overrides — never repeat `virtual`

## Step 6: Fix errors file by file

For each file with errors:
1. Read the full file to understand context
2. Apply all fixes for that file in one edit
3. Do not reformat unrelated code — only change what tidy flagged

## Step 7: Verify fixes

Re-run the same tidy command from Step 4 to confirm all errors are resolved. If new errors appear (e.g. from a fix that introduced another violation), fix those too and repeat until clean.

## Step 8: Report

Summarize the fixes made: which files were changed and which checks were resolved.
