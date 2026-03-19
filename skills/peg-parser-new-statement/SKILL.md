---
name: peg-parser-new-statement
description: Add a new SQL statement type to DuckDB's PEG parser. Use when the user asks to add, implement, or support parsing for a new SQL statement or syntax (e.g. CREATE TRIGGER, DROP TRIGGER, CREATE MATERIALIZED VIEW) in the PEG parser.
argument-hint: "[statement-name, e.g. CREATE TRIGGER]"
---

# Add a New Statement to the PEG Parser

Add `$ARGUMENTS` to DuckDB's PEG parser. The PEG parser lives in `extension/autocomplete/` and runs as a parser override extension before the legacy Postgres parser.

**Choose your path** based on the statement form:
- **CREATE-form** (e.g. `CREATE TRIGGER`, `CREATE MATERIALIZED VIEW`) → follow Shared Steps, then Part A
- **DROP-form** (e.g. `DROP TRIGGER`) → follow Shared Steps, then Part B

---

## Shared Step 1 — Add the catalog type

In `src/include/duckdb/common/enums/catalog_type.hpp`, add:

```cpp
XXX_ENTRY = <next available number>,
```

In `src/common/enums/catalog_type.cpp`, add to both `CatalogTypeToString` and `CatalogTypeFromString`.

---

## Shared Step 2 — Add enums (if needed)

Create `src/include/duckdb/common/enums/xxx_type.hpp` for any new enums specific to this statement, then regenerate:

```bash
python3 scripts/generate_enum_util.py
```

---

## Part A: CREATE-form statements

### A1 — Define the grammar

Create `extension/autocomplete/grammar/statements/create_xxx.gram`.

Wire the new statement into `CreateStatementVariation` in `create_table.gram`:

```
CreateStatementVariation <- ... / CreateXxxStmt
```

After any `.gram` edit, regenerate (never edit `inlined_grammar.gram` or `inlined_grammar.hpp` directly):

```bash
python3 extension/autocomplete/inline_grammar.py --grammar-file
python3 extension/autocomplete/inline_grammar.py
```

See the **Grammar reference** section below for syntax rules.

---

### A2 — Create the parsed data struct

**Header**: `src/include/duckdb/parser/parsed_data/create_xxx_info.hpp`

```cpp
#pragma once
#include "duckdb/parser/parsed_data/create_info.hpp"

namespace duckdb {

struct CreateXxxInfo : public CreateInfo {
    CreateXxxInfo();

    string name;
    // ... your fields

    unique_ptr<CreateInfo> Copy() const override;
    string ToString() const override;
    DUCKDB_API void Serialize(Serializer &serializer) const override;
    DUCKDB_API static unique_ptr<CreateInfo> Deserialize(Deserializer &deserializer);
};

} // namespace duckdb
```

**Implementation**: `src/parser/parsed_data/create_xxx_info.cpp`
- Constructor: `CreateXxxInfo() : CreateInfo(CatalogType::XXX_ENTRY, INVALID_SCHEMA) {}`
- `Copy()`: call `CopyProperties(*result)` then copy each field
- `ToString()`: reconstruct the SQL string

If a field is a `unique_ptr<SQLStatement>` (not `SelectStatement`), it cannot be auto-serialized. Store the SQL text alongside it:

```cpp
string sql_body_text;          // serialized
unique_ptr<SQLStatement> sql_body; // runtime only
```

Add to `src/parser/parsed_data/CMakeLists.txt`:
```cmake
create_xxx_info.cpp
```

---

### A3 — Add serialization config

In `src/include/duckdb/storage/serialization/create_info.json`, add:

```json
{
  "class": "CreateXxxInfo",
  "base": "CreateInfo",
  "enum": "XXX_ENTRY",
  "includes": ["duckdb/parser/parsed_data/create_xxx_info.hpp"],
  "members": [
    { "id": 200, "name": "field_name", "type": "string" },
    { "id": 201, "name": "my_enum",    "type": "MyEnumType" },
    { "id": 202, "name": "my_bool",    "type": "bool" }
  ]
}
```

Supported types: `string`, `bool`, `int64_t`, `uint64_t`, `vector<string>`, `unique_ptr<SelectStatement>`, any registered enum. `SQLStatement*` is **not** supported — use `string` instead.

```bash
python3 scripts/generate_serialization.py
```

---

### A4 — Write the transformer

Create `extension/autocomplete/transformer/transform_create_xxx.cpp`.

See the **Transformer patterns** section below for how to handle indexing, choices, optionals, lists, enums, and nested statements.

Add to `extension/autocomplete/transformer/CMakeLists.txt`:
```cmake
transform_create_xxx.cpp
```

---

### A5 — Register the transformer

In `extension/autocomplete/transformer/peg_transformer_factory.cpp`:

1. Add a `RegisterCreateXxx()` function and call it from the constructor (keep alphabetical order):
```cpp
void PEGTransformerFactory::RegisterCreateXxx() {
    REGISTER_TRANSFORM(TransformCreateXxxStmt);
    // ... all sub-rules
}
```
2. Add new enum mappings to `RegisterEnums()`.

In `extension/autocomplete/include/transformer/peg_transformer.hpp`:

1. Add `void RegisterCreateXxx();` with the other `RegisterCreate*` declarations.
2. Add a `static` declaration for each transform function.
3. If you have a custom intermediate struct, create it in `extension/autocomplete/include/ast/xxx_info.hpp` and include it from the header.

---

### A6 — Handle excluded rules

Rules used only as boolean flags (e.g. `ForEachRow <- 'FOR' 'EACH' 'ROW'`, checked via `HasResult()`) don't need a transformer. Add them to `EXCLUDED_RULES` in `scripts/generate_peg_transformer.py` to suppress coverage warnings.

---

### A7 — Add binder stub

In `src/planner/binder/statement/bind_create.cpp`, add before the `default` throw:

```cpp
case CatalogType::XXX_ENTRY:
    throw NotImplementedException("CREATE XXX is not yet supported");
```

---

### A8 — Write tests

Create `test/sql/xxx/create_xxx_parse.test`. Required boilerplate:

```sql
require autocomplete

statement ok
call enable_peg_parser();
```

**Syntax error tests must match a specific error message** — an empty `----` matches any error, including "not yet supported", so it won't catch a parser that silently accepted malformed input:

```sql
# WRONG — passes even if the parser didn't reject the bad syntax:
statement error
CREATE XXX bad syntax;
----

# RIGHT — proves the parser caught it:
statement error
CREATE XXX bad syntax;
----
Parser Error: Syntax error at or near "bad"
```

To find the exact error message, run the statement through the CLI:
```bash
echo "call enable_peg_parser(); CREATE XXX bad syntax;" | build/debug/duckdb
```

---

### A9 — Verify coverage and format

```bash
# All rules must be FOUND, ENUM, or EXCLUDED — no MISSING
python3 scripts/generate_peg_transformer.py | grep -A30 "File: create_xxx"

# Reformat changed files
make format-main
```

---

## Part B: DROP-form statements

DROP statements do **not** need a new parsed data struct. All drops share the single generic `DropInfo` — the catalog type set on it distinguishes which object is being dropped.

### B1 — Define the grammar

Create `extension/autocomplete/grammar/statements/drop_xxx.gram`.

Wire the new statement into `DropStatementVariation` in `drop_table.gram`:

```
DropStatementVariation <- ... / DropXxxStmt
```

After any `.gram` edit, regenerate:

```bash
python3 extension/autocomplete/inline_grammar.py --grammar-file
python3 extension/autocomplete/inline_grammar.py
```

See the **Grammar reference** section below for syntax rules.

---

### B2 — Write the transformer

Create `extension/autocomplete/transformer/transform_drop_xxx.cpp`. Produce a `DropInfo` — **not** a new struct:

```cpp
#include "duckdb/parser/parsed_data/drop_info.hpp"

static unique_ptr<ParsedExpression> TransformDropXxxStmt(PEGTransformer &transformer, ParseResult &pr) {
    auto &list_pr = pr.Cast<ListParseResult>();
    // 'DROP' 'XXX' IfExists? QualifiedName
    //   [0]   [1]    [2]       [3]
    auto result = make_uniq<DropInfo>();
    result->type = CatalogType::XXX_ENTRY;
    result->if_exists = list_pr.Child<OptionalParseResult>(2).HasResult();
    auto qname = transformer.Transform<QualifiedName>(list_pr.Child<ListParseResult>(3));
    result->name = qname.name;
    result->schema = qname.schema;
    return result;
}
```

See the **Transformer patterns** section below for handling other grammar constructs.

Add to `extension/autocomplete/transformer/CMakeLists.txt`:
```cmake
transform_drop_xxx.cpp
```

---

### B3 — Register the transformer

In `extension/autocomplete/transformer/peg_transformer_factory.cpp`:

1. Add a `RegisterDropXxx()` function and call it from the constructor (keep alphabetical order):
```cpp
void PEGTransformerFactory::RegisterDropXxx() {
    REGISTER_TRANSFORM(TransformDropXxxStmt);
    // ... all sub-rules
}
```

In `extension/autocomplete/include/transformer/peg_transformer.hpp`:

1. Add `void RegisterDropXxx();` with the other `RegisterDrop*` declarations.
2. Add a `static` declaration for each transform function.

---

### B4 — Handle excluded rules

Same as A6 — add boolean-flag-only rules to `EXCLUDED_RULES` in `scripts/generate_peg_transformer.py`.

---

### B5 — Add binder stub

In `src/planner/binder/statement/bind_drop.cpp`, add before the `default` throw:

```cpp
case CatalogType::XXX_ENTRY:
    throw NotImplementedException("DROP XXX is not yet supported");
```

---

### B6 — Write tests

Create `test/sql/xxx/drop_xxx_parse.test`. Required boilerplate:

```sql
require autocomplete

statement ok
call enable_peg_parser();
```

Syntax error tests must match a specific error message (see A8 for the rationale and pattern).

To find the exact error message:
```bash
echo "call enable_peg_parser(); DROP XXX bad syntax;" | build/debug/duckdb
```

---

### B7 — Verify coverage and format

```bash
# All rules must be FOUND, ENUM, or EXCLUDED — no MISSING
python3 scripts/generate_peg_transformer.py | grep -A30 "File: drop_xxx"

# Reformat changed files
make format-main
```

---

## Grammar reference

Key grammar syntax:
- `'KEYWORD'` — case-insensitive literal
- `Rule1 / Rule2` — ordered choice (first match wins)
- `Rule?` — optional, `Rule*` / `Rule+` — repeat
- `List(D)` — comma-separated list (defined in `common.gram`)
- `Parens(D)` — `D` wrapped in parentheses
- `Statement` — any full DuckDB SQL statement
- `QualifiedName` — `[catalog.]schema.name` identifier
- `IfNotExists` — already defined, no transformer needed
- `IfExists` — already defined, no transformer needed

**Critical**: never use `List(...)` directly inline in a rule — it has no transformer and will crash at runtime. Always wrap it in a named rule:

```
# WRONG:
MyRule <- 'STUFF' List(ColId)

# RIGHT:
MyRule <- 'STUFF' MyColumnList
MyColumnList <- List(ColId)
```

---

## Transformer patterns

**Naming contract**: `REGISTER_TRANSFORM` strips the `"Transform"` prefix (9 chars) and registers the remainder as the rule name key. So `TransformFooBar` must match grammar rule `FooBar` exactly — a mismatch causes a runtime crash.

**Indexing**: every element in a grammar rule — keywords, optionals, sub-rules — is a child at a fixed 0-based index:
```
CreateXxxStmt <- 'XXX' IfNotExists? QualifiedName ...
                  [0]   [1]          [2]
```

**Choices** (`/`): use `ChoiceParseResult` — its `.result` carries the matched rule's name:
```cpp
return transformer.Transform<MyType>(list_pr.Child<ChoiceParseResult>(0).result);
```

**Optionals** (`?`): use `OptionalParseResult::HasResult()`:
```cpp
bool if_not_exists = list_pr.Child<OptionalParseResult>(1).HasResult();
```

**Lists**: use `ExtractParseResultsFromList` then loop:
```cpp
auto items = ExtractParseResultsFromList(list_pr.Child<ListParseResult>(0));
for (auto &item : items) {
    result.push_back(transformer.Transform<string>(item));
}
```

**Enums**: register in `RegisterEnums()`, then use `TransformEnum<T>`:
```cpp
// registration:
RegisterEnum<MyEnum>("RuleName", MyEnum::VALUE);
// transformer:
return transformer.TransformEnum<MyEnum>(list_pr.Child<ChoiceParseResult>(0).result);
```

**Nested statement**:
```cpp
auto body = transformer.Transform<unique_ptr<SQLStatement>>(list_pr.Child<ListParseResult>(N));
```

---

## Reference implementations

- **CreateView** — simplest CREATE-form: `create_view.gram` + `transform_create_view.cpp`
- **CreateSequence** — enum options + repeated sub-rules: `create_sequence.gram` + `transform_create_sequence.cpp`
- **CreateTrigger** — choices, column list, nested statement body: `create_trigger.gram` + `transform_create_trigger.cpp`
