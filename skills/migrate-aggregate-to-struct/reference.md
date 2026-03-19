# C++ Type to LogicalType Mapping

Use this table when creating the `GetExportStateType` callback.

| C++ Type | LogicalType |
|----------|-------------|
| `bool` | `LogicalType::BOOLEAN` |
| `int8_t` | `LogicalType::TINYINT` |
| `int16_t` | `LogicalType::SMALLINT` |
| `int32_t` | `LogicalType::INTEGER` |
| `int64_t` | `LogicalType::BIGINT` |
| `uint8_t` | `LogicalType::UTINYINT` |
| `uint16_t` | `LogicalType::USMALLINT` |
| `uint32_t` | `LogicalType::UINTEGER` |
| `uint64_t` | `LogicalType::UBIGINT` |
| `hugeint_t` | `LogicalType::HUGEINT` |
| `uhugeint_t` | `LogicalType::UHUGEINT` |
| `float` | `LogicalType::FLOAT` |
| `double` | `LogicalType::DOUBLE` |
| `string_t` | `LogicalType::VARCHAR` |
| `interval_t` | `LogicalType::INTERVAL` |

## Dynamic types

When the state struct is templated (e.g. `BitState<T>`) and the concrete type depends on input:

- Use `function.return_type` if the state's value field matches the return type
- Use `function.arguments[0]` / `function.arguments[1]` if it matches an argument type

## Supported physical types for serialization

The `TemplateDispatch` in `src/function/scalar/system/aggregate_export.cpp` handles these PhysicalTypes: BOOL, UINT8, UINT16, UINT32, UINT64, UINT128, INT8, INT16, INT32, INT64, INT128, FLOAT, DOUBLE, VARCHAR, INTERVAL, and STRUCT (recursively).

Any LogicalType that maps to one of these physical types is supported.
