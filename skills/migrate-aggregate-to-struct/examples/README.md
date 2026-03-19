# Migration Examples

These are real examples from the DuckDB codebase of aggregate functions that have already been migrated to struct-based state export.

## Simple: product (flat struct, fixed types)

**State struct** (`extension/core_functions/aggregate/distributive/product.cpp`):
```cpp
struct ProductState {
    bool empty;
    double val;
};
```

**GetExportStateType callback**:
```cpp
LogicalType GetProductStateType(const AggregateFunction &function) {
    child_list_t<LogicalType> children;
    children.emplace_back("empty", LogicalType::BOOLEAN);
    children.emplace_back("val", LogicalType::DOUBLE);
    return LogicalType::STRUCT(std::move(children));
}
```

**Registration**:
```cpp
AggregateFunction ProductFun::GetFunction() {
    return AggregateFunction::UnaryAggregate<ProductState, double, double, ProductFunction>(
               LogicalType(LogicalTypeId::DOUBLE), LogicalType::DOUBLE)
        .SetStructStateExport(GetProductStateType);
}
```

## Template-parameterized: min/max (type depends on input)

**State struct** (`src/function/aggregate/distributive/minmax.cpp`):
```cpp
template <class T>
struct MinMaxState {
    T value;
    bool isset;
};
```

**GetExportStateType callback** — uses `function.return_type` since `T` varies:
```cpp
LogicalType GetExportStateType(const AggregateFunction &function) {
    auto struct_children_types = child_list_t<LogicalType> {};
    struct_children_types.emplace_back("value", function.return_type);
    struct_children_types.emplace_back("isset", LogicalType::BOOLEAN);
    return LogicalType::STRUCT(std::move(struct_children_types));
}
```

**Registration** — applied to each function in the set:
```cpp
AggregateFunctionSet MaxFun::GetFunctions() {
    AggregateFunctionSet max("max");
    max.AddFunction(MaxFunction::GetFunction().SetStructStateExport(GetExportStateType));
    max.AddFunction(GetMinMaxNFunction<GreaterThan>().SetStructStateExport(GetExportStateType));
    return max;
}
```

## Nested structs: corr (state contains other aggregate states)

**State struct** (`extension/core_functions/aggregate/algebraic/corr.hpp`):
```cpp
struct CorrState {
    CovarState cov_pop;
    StddevState dev_pop_x;
    StddevState dev_pop_y;
};
```

Where `CovarState` has `{uint64_t count, double mean_x, double mean_y, double co_moment}` and `StddevState` has `{uint64_t count, double mean, double dsquared}`.

**GetExportStateType callback** — builds nested STRUCT types:
```cpp
LogicalType GetCorrExportStateType(const AggregateFunction &function) {
    auto state_children = child_list_t<LogicalType> {};

    auto covar_pop_children = child_list_t<LogicalType> {};
    covar_pop_children.emplace_back("count", LogicalType::UBIGINT);
    covar_pop_children.emplace_back("mean_x", LogicalType::DOUBLE);
    covar_pop_children.emplace_back("mean_y", LogicalType::DOUBLE);
    covar_pop_children.emplace_back("co_moment", LogicalType::DOUBLE);
    state_children.emplace_back("cov_pop", LogicalType::STRUCT(std::move(covar_pop_children)));

    auto dev_pop_x_children = child_list_t<LogicalType> {};
    dev_pop_x_children.emplace_back("count", LogicalType::UBIGINT);
    dev_pop_x_children.emplace_back("mean", LogicalType::DOUBLE);
    dev_pop_x_children.emplace_back("dsquared", LogicalType::DOUBLE);
    state_children.emplace_back("dev_pop_x", LogicalType::STRUCT(std::move(dev_pop_x_children)));

    auto dev_pop_y_children = child_list_t<LogicalType> {};
    dev_pop_y_children.emplace_back("count", LogicalType::UBIGINT);
    dev_pop_y_children.emplace_back("mean", LogicalType::DOUBLE);
    dev_pop_y_children.emplace_back("dsquared", LogicalType::DOUBLE);
    state_children.emplace_back("dev_pop_y", LogicalType::STRUCT(std::move(dev_pop_y_children)));

    return LogicalType::STRUCT(std::move(state_children));
}
```

## Function set with mixed eligibility: bit_and (some overloads have destructors)

**State struct** (`extension/core_functions/aggregate/distributive/bitagg.cpp`):
```cpp
template <class T>
struct BitState {
    using TYPE = T;
    bool is_set;
    T value;
};
```

The integral overloads are eligible, but the BIT (string_t) overload uses `UnaryAggregateDestructor` and must be skipped:

```cpp
AggregateFunctionSet BitAndFun::GetFunctions() {
    AggregateFunctionSet bit_and;
    for (auto &type : LogicalType::Integral()) {
        bit_and.AddFunction(
            GetBitfieldUnaryAggregate<BitAndOperation>(type)
                .SetStructStateExport(GetBitStateType));  // eligible
    }
    // BIT-type overload has a destructor — do NOT add SetStructStateExport
    bit_and.AddFunction(
        AggregateFunction::UnaryAggregateDestructor<BitState<string_t>, string_t, string_t, BitStringAndOperation>(
            LogicalType::BIT, LogicalType::BIT));
    return bit_and;
}
```
