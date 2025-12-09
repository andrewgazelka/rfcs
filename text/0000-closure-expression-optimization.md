- Feature Name: closure_expression_optimization
- Start Date: 2025-12-08
- RFC PR: [nushell/rfcs#8](https://github.com/nushell/rfcs/pull/8)
- Nushell Issue: [nushell/nushell#0000](https://github.com/nushell/nushell/issues/0000)

# Summary

Enable nushell to analyze and optimize closures passed to commands like `where`, `select`, and `sort-by` by inspecting their AST structure. This allows optimizations such as predicate pushdown to data sources (databases, Polars DataFrames) and reordering of pipeline stages.

# Motivation

Currently, closures in nushell are opaque to the runtime. When you write:

```nushell
open data.db | query "SELECT * FROM users" | where { $in.age > 30 }
```

Nushell must fetch all rows and filter them in-memory, even though the filter could be pushed to the database as `WHERE age > 30`.

Similarly, with Polars:

```nushell
polars open large.parquet | polars into-nu | where { $in.status == "active" }
```

The entire parquet file is loaded into memory before filtering, rather than letting Polars apply the predicate during scan.

SQL optimizers have done predicate pushdown for decades because `σ(A ⋈ B) ≡ σ(A) ⋈ σ(B)` when the predicate only touches A's columns. Nushell could benefit from the same optimization if closures were analyzable.

The key insight is that nushell already parses closures into an AST—they aren't compiled to opaque bytecode like Python lambdas. The infrastructure exists; we just need to build the analysis pass.

# Guide-level explanation

## What changes for users

Nothing changes syntactically. Users write the same closures they always have. The difference is that certain patterns will execute faster because nushell can optimize them.

For example, this pipeline:

```nushell
polars open sales.parquet | polars into-nu | where { $in.year == 2024 } | where { $in.amount > 1000 }
```

Could be automatically optimized to push both predicates to Polars, equivalent to:

```nushell
polars open sales.parquet | polars filter ((polars col year) == 2024) | polars filter ((polars col amount) > 1000) | polars into-nu
```

## Optimizable vs non-optimizable closures

Not all closures can be optimized. The optimizer recognizes specific patterns:

**Optimizable:**
```nushell
{ $in.age > 30 }                      # Simple comparison
{ $in.name == "Alice" }               # Equality check
{ $in.age > 30 and $in.active }       # Logical combinations
{ $in.score >= $threshold }           # Captured variables (evaluated first)
{ $in.name | str starts-with "A" }    # Known-pure string operations
```

**Not optimizable (falls back to normal execution):**
```nushell
{ $in.age > (expensive_computation) } # Arbitrary subexpressions
{ $in | custom-command }              # Unknown command purity
{ mut x = 0; $x += $in.val; $x }      # Mutation
{ print $in.name; $in.age > 30 }      # Side effects
```

When a closure cannot be optimized, it executes exactly as it does today—no behavior change, just no optimization.

## Inspecting optimization decisions

A new flag could be added to show optimization decisions:

```nushell
> open db.sqlite | query "SELECT * FROM users" | where { $in.age > 30 } --explain
# Predicate `$in.age > 30` pushed to SQL: WHERE age > 30
```

# Reference-level explanation

## Closure analysis

The optimizer walks the closure's AST and attempts to extract a "predicate expression" that can be translated to the target system. The analysis proceeds as follows:

1. **Entry point check**: Verify the closure body is a single expression (not a block with multiple statements).

2. **Expression classification**: Recursively classify each AST node:
   - `BinaryOp(==, !=, <, >, <=, >=)` with field access and literal → `Translatable`
   - `BinaryOp(and, or)` with translatable children → `Translatable`
   - `UnaryOp(not)` with translatable child → `Translatable`
   - Field access on `$in` → `FieldRef`
   - Literal values → `Literal`
   - Captured variable reference → evaluate eagerly, treat as `Literal`
   - Known-pure commands (e.g., `str starts-with`) → `Translatable` if args are translatable
   - Anything else → `Opaque`

3. **Purity verification**: Confirm no side effects exist in the expression tree. This requires maintaining a registry of pure commands.

4. **Translation**: Convert the classified AST to the target representation (SQL WHERE clause, Polars expression, etc.).

## Pure command registry

A new registry tracks which commands are pure (no side effects, deterministic output for same input):

```rust
pub struct PurityRegistry {
    pure_commands: HashSet<String>,
}

impl PurityRegistry {
    pub fn is_pure(&self, cmd: &str) -> bool {
        self.pure_commands.contains(cmd)
    }
}
```

Initial pure commands would include: `str length`, `str starts-with`, `str ends-with`, `str contains`, `str trim`, `math abs`, `math round`, etc.

## Integration points

### Database sources

When `open` or `query` detects a downstream `where` with a translatable predicate:

```
Pipeline: [SqlSource, Where(closure)]
  → [SqlSource(with_where_clause)]
```

### Polars plugin

The Polars plugin could expose an optimization hook:

```rust
trait OptimizableSource {
    fn accepts_predicate(&self, pred: &PredicateExpr) -> bool;
    fn push_predicate(&mut self, pred: PredicateExpr);
}
```

### Pipeline reordering

Beyond pushdown, the optimizer could reorder operations:

```nushell
# Before: sort entire dataset, then filter
ls | sort-by size | where { $in.size > 1mb }

# After: filter first, sort smaller dataset
ls | where { $in.size > 1mb } | sort-by size
```

This is valid when the predicate doesn't depend on sort order.

## Captured variables

Captured variables must be evaluated before predicate translation:

```nushell
let threshold = 30
where { $in.age > $threshold }
# Translates to: WHERE age > 30 (not WHERE age > threshold)
```

The optimizer evaluates `$threshold` at optimization time and substitutes the concrete value.

## Escape hatch

If optimization causes issues, users can force opaque execution:

```nushell
where {|| $in.age > 30 }  # Double-pipe signals "don't optimize"
```

Or a command flag:

```nushell
where --no-optimize { $in.age > 30 }
```

# Drawbacks

1. **Complexity**: Adds significant complexity to the pipeline execution model. More code to maintain, more edge cases to handle.

2. **Semantic subtlety**: Optimization changes *when* code runs. A closure like `{ $in.age > (rand) }` would behave differently if the `rand` call were hoisted vs evaluated per-row. The optimizer must be conservative.

3. **Debugging difficulty**: When predicates are pushed down, error messages and stack traces may be confusing—the error occurs in the database, not in nushell.

4. **Limited applicability**: Many nushell users work with small datasets where optimization overhead exceeds benefits.

5. **Pure command maintenance**: The purity registry must be maintained as commands are added/modified.

# Rationale and alternatives

## Why AST analysis over expression sublanguage

**Alternative**: Create a restricted expression DSL separate from closures (like Polars does).

**Rationale against**: This fragments the language. Users would need to learn when to use closures vs expressions. AST analysis keeps the syntax unified—write closures everywhere, get optimization where possible.

## Why not JIT compilation

**Alternative**: JIT compile closures and use runtime profiling to guide optimization.

**Rationale against**: Massive implementation complexity. JIT requires platform-specific code generation, sophisticated runtime infrastructure. AST analysis is simpler and portable.

## Why not type-directed optimization

**Alternative**: Use a richer type system (algebraic effects, purity types) to determine what's optimizable.

**Rationale against**: Would require significant language changes. AST pattern matching is pragmatic and doesn't change the surface language.

## Impact of not doing this

Nushell remains slower than it could be for database and dataframe workloads. Users working with large datasets continue to need manual optimization (writing SQL directly, using Polars expressions explicitly).

# Prior art

## LINQ (C#)

LINQ's `Expression<Func<T, bool>>` captures lambdas as expression trees rather than compiled delegates. This enables Entity Framework to translate:

```csharp
users.Where(u => u.Age > 30)
```

to SQL. Nushell's situation is analogous—we have ASTs, we just don't exploit them.

## Spark SQL

Spark DataFrames build logical plans from operations. User-defined functions (UDFs) break optimization because they're opaque. Spark explicitly warns about this and provides "Pandas UDFs" with limited semantics for partial optimization.

## jq

jq is a pure, total language for JSON transformation. Its restricted semantics make optimization safe—no side effects, guaranteed termination. Nushell closures are more powerful but could identify a "jq-like subset" that's optimizable.

## Polars lazy expressions

Polars expressions (`pl.col("age") > 30`) are designed for optimization from the start. They're not general-purpose code—they're a DSL. Nushell's approach would be to recognize when closures happen to match this DSL's semantics.

# Unresolved questions

1. **Purity granularity**: Should purity be per-command or per-invocation? `http get` is impure, but `str length` is pure. What about `random int` (pure but non-deterministic)?

2. **Optimization visibility**: How do users know when optimization occurred? Silent optimization is convenient but makes debugging harder.

3. **Cross-plugin protocol**: How do plugins advertise optimization capabilities? A new protocol message type?

4. **Partial pushdown**: If a predicate is `$in.age > 30 and (complex_thing)`, can we push the first part and keep the second?

5. **Correctness testing**: How do we ensure optimized and unoptimized paths produce identical results?

# Future possibilities

## Projection pushdown

Beyond predicates, push column selection:

```nushell
open db.sqlite | query "SELECT * FROM users" | select name age
# → SELECT name, age FROM users
```

## Join optimization

Analyze multiple data sources and optimize join order:

```nushell
$users | join $orders --on id | where { $in.total > 100 }
# Could push predicate to orders table before join
```

## Cost-based optimization

With statistics about data sources (row counts, index availability), make smarter decisions about when pushdown helps vs hurts.

## User-defined purity annotations

Allow users to mark custom commands as pure:

```nushell
def my-transform [x] --pure {
    $x | str upcase | str trim
}
```

## Incremental/streaming optimization

For streaming data sources, maintain optimizer state across chunks to enable cross-chunk optimizations.
