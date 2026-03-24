# DakodaStemen/zio-blocks

## About This Fork

This is a fork of [zio-org/zio-blocks](https://github.com/zio/zio-blocks), maintained by DakodaStemen as part of a bounty contribution to the ZIO Blocks open-source project. The contribution focuses on implementing the schema migration infrastructure for the `schema` module — a capability that allows typed, reversible, serializable migrations between successive versions of data types.

The work lives on the `schema-migration` branch and is being prepared for upstream submission.

---

## Schema Migration Contribution

### Motivation

In any system where data is persisted — event stores, document databases, message queues — the shape of data evolves over time. Adding fields, renaming them, changing types, splitting or joining values: all of these are routine, but handling them safely and reversibly requires a principled approach.

ZIO Blocks Schema already provides a powerful reflection layer via `DynamicValue` and `DynamicOptic`. The missing piece was a first-class migration API: a way to describe structural transformations between schema versions, execute them against `DynamicValue` at runtime, and compose them into upgrade paths that can be reversed or chained.

This contribution adds that layer.

---

### What Was Implemented

The following files were introduced across two substantive commits (`9eac545` and `0abe1d1`), with a final sbt integration pass in `0fa2411`:

#### `DynamicMigration.scala`

The untyped, serializable core. A `DynamicMigration` is a `Chunk[MigrationAction]` that operates on `DynamicValue`. It is fully serializable — no closures, no reflection at runtime. Key properties:

- `apply(value: DynamicValue): Either[MigrationError, DynamicValue]` — executes the migration as a left fold over the action list
- `reverse: DynamicMigration` — produces the structural inverse by reversing the action sequence and inverting each step
- `++(other): DynamicMigration` — composes migrations sequentially; composition is associative

#### `MigrationAction.scala` (inline in `DynamicMigration`)

An ADT of all supported migration operations, each addressed via `DynamicOptic`:

| Action | Description |
|--------|-------------|
| `AddField` | Insert a new field at a path, computed from a `MigrationExpr` |
| `DropField` | Remove a field; carries a default for reverse (re-add) |
| `RenameField` | Move a field value from one path to another |
| `TransformValue` | Apply a `MigrationExpr` to the value at a path; carries the inverse expression |
| `Optionalize` | Wrap a field value in `Some`; inverse is `Mandate` |
| `Mandate` | Unwrap `Some` from an optional field; fails on `None` |
| `Join` | Merge two fields into one via a combining expression; inverse is `Split` |
| `Split` | Decompose a field into two via projection expressions; inverse is `Join` |
| `RenameCase` | Rename a variant case by label in a `DynamicValue.Variant` |
| `ChangeFieldType` | Apply a type-conversion expression with its inverse |

Every action carries enough information to invert itself. `DynamicMigration.reverse` is therefore a pure structural operation with no external state.

#### `MigrationExpr.scala`

A serializable AST for value-to-value transformations used by migration actions. No user lambdas are admitted; all expressions are data:

- `Literal(value)` — constant
- `IntToString` / `StringToInt` — primitive coercions
- `StringConcat(separator)` — combines `_left` and `_right` fields of a record (used with `Join`)
- `StringSplitLeft(sep)` / `StringSplitRight(sep)` — decompose a string (used with `Split`)

This design ensures migrations can be persisted, transmitted, and replayed offline without any reference to application code.

#### `Migration.scala`

The typed, user-facing wrapper:

```scala
final case class Migration[A, B](
  sourceSchema: Schema[A],
  targetSchema: Schema[B],
  underlying: DynamicMigration
) {
  def apply(a: A): Either[MigrationError, B]
  def reverse: Migration[B, A]
  def andThen[C](other: Migration[B, C]): Migration[A, C]
}
```

`Migration[A, B]` converts `A` to `DynamicValue` using `sourceSchema`, threads it through `DynamicMigration`, then decodes back to `B` using `targetSchema`. The typed layer is thin; the migration logic lives entirely in the untyped core.

#### `MigrationBuilder.scala`

A fluent builder for constructing `Migration[A, B]` values by appending `MigrationAction`s one at a time:

```scala
val migration =
  MigrationBuilder[PersonV1, PersonV2]
    .renameField(DynamicOptic.root.field("name"), DynamicOptic.root.field("fullName"))
    .build
```

The `build` method requires implicit `Schema[A]` and `Schema[B]` in scope and wraps the accumulated actions in a `Migration`.

#### `MigrationBuilderVersionSpecific.scala` (Scala 2 and Scala 3)

A version-specific mixin trait that overloads `MigrationBuilder` methods to accept selector lambdas instead of `DynamicOptic` values directly:

```scala
// Scala 2 — resolved at compile time via blackbox macro
builder.renameField(_.name, _.fullName)
```

In Scala 2, the implementation uses `scala.reflect.macros.blackbox`. The macro (`MigrationMacros.selectorBodyToOptic`) walks the lambda body AST, collects field-access steps via recursive `q"$parent.$field"` pattern matching, and synthesises a `DynamicOptic` tree. In Scala 3, the equivalent uses `scala.quoted.*` with `quotes.reflect`, following the same field-select chain extraction pattern adapted to the new macro API.

The selector API is pure ergonomic sugar; the underlying `DynamicOptic`-based methods in `MigrationBuilder` are always available as the primary interface.

#### `MigrationMacros.scala` (Scala 2 and Scala 3)

Implements the compile-time selector-to-optic translation described above. The Scala 2 version is a `blackbox.Context` macro object; the Scala 3 version uses `inline def` with `Expr`-based quotes. Both reject non-lambda arguments and non-field-select bodies with a compiler error at the call site.

#### `MigrationError.scala`

```scala
final case class MigrationError(message: String, at: DynamicOptic)
  extends Exception with NoStackTrace
```

All failures carry the `DynamicOptic` path where the error occurred, enabling precise diagnostics.

---

### Test Coverage

Two test suites were added:

**`MigrationSpec`** — integration tests verifying:
- `DynamicMigration.empty` identity laws (left and right identity, action-list preservation)
- Associativity of `++`
- Structural reverse: `m.reverse.reverse` has the same action count as `m`
- Round-trip correctness for `RenameField`, `AddField`/`DropField`, `TransformValue` (Int/String coercion), `RenameCase`
- Typed `Migration[A, B]` identity for `Int` and `String`
- End-to-end typed round-trip: `PersonV1 -> PersonV2` via a rename, using `Schema.derived`

**`MigrationLawsSpec`** — property-based law tests (449 lines) using ZIO Test generators:
- Generator combinators for flat records, primitive values, and field names
- Property tests for all action types against randomly generated `DynamicValue.Record` inputs
- Round-trip laws: `m.reverse(m(v)) == v` for each action type where applicable
- Generators for `Join`/`Split` scenarios requiring records with exactly two string-valued fields

---

### Design Notes

**Serializable by construction.** The absence of closures in `MigrationExpr` and `MigrationAction` means the full migration plan is a plain Scala value that can be encoded to any format ZIO Blocks Schema supports. This is a deliberate constraint: migrations that reference application lambdas cannot survive process boundaries.

**Reversibility is structural.** Every `MigrationAction` carries the information needed to invert it. `DynamicMigration.reverse` does not require any external registry or schema introspection — it is a pure transformation of the action list.

**Typed API is thin.** `Migration[A, B]` adds schema-aware encode/decode on top of the untyped core. The migration logic itself never touches `Schema[A]` or `Schema[B]`; it operates on `DynamicValue` throughout. This keeps the core portable and independently testable.

**Cross-version macro strategy.** Supporting Scala 2 and Scala 3 selector ergonomics required two distinct macro implementations. The Scala 2 approach uses the established `blackbox.Context` API with quasiquotes; the Scala 3 approach uses `scala.quoted.*` with `quotes.reflect`. Both share the same semantic contract: a lambda `_.a.b` yields `DynamicOptic.root.field("a").field("b")`.

---

### Branch Reference

| Branch | Description |
|--------|-------------|
| `schema-migration` | Active contribution branch: schema migration infrastructure |
| `main` | Tracks upstream `zio-org/zio-blocks` main |

To review the contribution:

```bash
git clone https://github.com/DakodaStemen/zio-blocks.git
cd zio-blocks
git checkout schema-migration
```

Relevant files:

```
schema/shared/src/main/scala/zio/blocks/schema/DynamicMigration.scala
schema/shared/src/main/scala/zio/blocks/schema/Migration.scala
schema/shared/src/main/scala/zio/blocks/schema/MigrationBuilder.scala
schema/shared/src/main/scala/zio/blocks/schema/MigrationError.scala
schema/shared/src/main/scala/zio/blocks/schema/MigrationExpr.scala
schema/shared/src/main/scala-2/zio/blocks/schema/MigrationBuilderVersionSpecific.scala
schema/shared/src/main/scala-2/zio/blocks/schema/MigrationMacros.scala
schema/shared/src/main/scala-3/zio/blocks/schema/MigrationBuilderVersionSpecific.scala
schema/shared/src/main/scala-3/zio/blocks/schema/MigrationMacros.scala
schema/shared/src/test/scala/zio/blocks/schema/MigrationSpec.scala
schema/shared/src/test/scala/zio/blocks/schema/MigrationLawsSpec.scala
```

---

## Upstream ZIO Blocks

This fork is derived from [zio-org/zio-blocks](https://github.com/zio/zio-blocks), a family of modular, zero-dependency building blocks for Scala applications. ZIO Blocks provides type-safe schemas with automatic codec derivation, high-performance immutable sequences, compile-time safe resource management, and more. Each block is a standalone artifact designed to work with any Scala effect system — ZIO, Cats Effect, or plain Scala — and supports both Scala 2.13 and Scala 3.x on JVM and Scala.js.

See the upstream repository for full documentation, available modules, and contribution guidelines.
