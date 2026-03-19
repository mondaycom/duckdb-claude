---
name: migrate-aggregate-to-struct
description: Migrate a DuckDB aggregate function from opaque EXPORT_STATE to struct-based EXPORT_STATE using SetStructStateExport
argument-hint: "[function-name]"
disable-model-invocation: true
---

# Migrate Aggregate Function to Struct-Based State Export

Migrate the `$ARGUMENTS` aggregate function from legacy opaque (binary blob) EXPORT_STATE to the new struct-based format using `SetStructStateExport`.

## Step 1: Locate the implementation

Find the aggregate function's source file. Common locations:
- `extension/core_functions/aggregate/distributive/` (bit aggregates, bool, approx_count, kurtosis, skew)
- `extension/core_functions/aggregate/algebraic/` (avg, stddev, corr, covar)
- `extension/core_functions/aggregate/regression/` (regr_*)
- `src/function/aggregate/distributive/` (count, first/last, min/max)

Read the full implementation file.

## Step 2: Verify eligibility

Check against the requirements in `src/function/scalar/system/aggregate_export.cpp` (the `ExportAggregateFunction::Bind` method). A function CANNOT use EXPORT_STATE if it has:
- A **custom binder** (`HasBindCallback()`) - e.g. arg_min/arg_max
- A **destructor** (`HasStateDestructorCallback()`) - e.g. string_agg, list, BIT-type variants using `UnaryAggregateDestructor`
- No **combine** callback

If the function has multiple overloads (e.g. one for integral types, one for BIT type with a destructor), only the eligible overloads can get `SetStructStateExport`. Skip the overload with a destructor.

**Stop and inform the user if the function is not eligible.**

## Step 3: Identify the state struct

Find the C++ struct that holds the aggregate state. Note the **exact field order and types** as declared — the LogicalType struct fields MUST match the C++ memory layout. For reference, read already-migrated examples in [examples/](examples/).

## Step 4: Create the GetExportStateType callback

Add a function that maps the C++ state struct to a `LogicalType::STRUCT`. Place it in the same `.cpp` file, inside the anonymous namespace if one exists, before the function registration.

Use the type mapping from [reference.md](reference.md) to convert C++ types to LogicalTypes.

If the state struct is templated and the concrete type depends on the input, use `function.return_type` or the appropriate argument type dynamically:

```cpp
LogicalType GetMyStateType(const AggregateFunction &function) {
    child_list_t<LogicalType> children;
    children.emplace_back("is_set", LogicalType::BOOLEAN);
    children.emplace_back("value", function.return_type);
    return LogicalType::STRUCT(std::move(children));
}
```

For nested struct states, create nested `LogicalType::STRUCT` values — see the `corr` example in [examples/](examples/).

## Step 5: Wire up SetStructStateExport

Chain `.SetStructStateExport(YourCallback)` onto each eligible `AggregateFunction` in the registration function (`GetFunction()` or `GetFunctions()`).

For function sets with multiple overloads, add it to each eligible overload individually. Do NOT add it to overloads that have destructors or bind callbacks.

## Step 6: Add tests to test_state_export_struct.test

Add the function to `test/sql/aggregate/aggregates/test_state_export_struct.test`. Follow the existing patterns:

1. Add to **ungrouped aggregate queries** (`res0` nosort group) — both the plain query and the `finalize(...EXPORT_STATE)` query
2. Add to **grouped aggregate queries** (`res1` nosort group) — with `GROUP BY g ORDER BY g`
3. Add to the **persisted state table** (`CREATE TABLE state AS ...`) and corresponding `finalize()` query
4. Add to **empty group tests** (`res4`), **null-scanned tests** (`res5`, `res6`)

Update the `query` type strings to reflect the new column count.

## Step 7: Add an EXPORT_STATE validation test

Verify the exported state values are correct by selecting the EXPORT_STATE result directly (without casting):

```sql
query T
SELECT your_func(42) EXPORT_STATE;
----
{'field1': value1, 'field2': value2}
```

Run the query first to get the actual output, then use that as the expected result.

## Step 8: Remove from opaque tests (if applicable)

If the function was tested in `test/sql/aggregate/aggregates/test_state_export_opaque.test`, remove it from there. The opaque test file should only contain functions that have NOT been migrated.

## Step 9: Build and run tests

```bash
cd build/debug && ninja -j8
./test/unittest "test/sql/aggregate/aggregates/test_state_export_struct.test"
```

## Checklist

- [ ] GetExportStateType field order matches C++ struct declaration order
- [ ] LogicalTypes match the C++ types exactly (see [reference.md](reference.md))
- [ ] SetStructStateExport chained on ALL eligible overloads (skip overloads with destructors/binders)
- [ ] Tests added to test_state_export_struct.test
- [ ] Tests removed from test_state_export_opaque.test (if applicable)
- [ ] Build succeeds and tests pass
