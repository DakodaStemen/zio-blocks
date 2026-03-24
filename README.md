# DakodaStemen/zio-blocks

A fork of [zio/zio-blocks](https://github.com/zio/zio-blocks), maintained as part of a bounty contribution to the ZIO Blocks open-source project. The contribution implements schema migration infrastructure for the `schema` module — a capability for typed, reversible, serializable migrations between successive versions of data types.

The work lives on the `schema-migration` branch and is being prepared for upstream submission.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Scala](https://img.shields.io/badge/Scala-2.13%20%2F%203.x-red.svg)](https://www.scala-lang.org/)

---

## Table of Contents

- [Motivation](#motivation)
- [What Was Implemented](#what-was-implemented)
- [DynamicMigration](#dynamicmigration)
- [MigrationAction ADT](#migrationaction-adt)
- [MigrationExpr](#migrationexpr)
- [TypedMigration](#typedmigration)
- [MigrationRegistry](#migrationregistry)
- [Design Decisions](#design-decisions)
- [Usage Example](#usage-example)
- [Running the Tests](#running-the-tests)
- [Upstream](#upstream)
- [License](#license)

---

## Motivation

In any system where data is persisted — event stores, document databases, message queues — the shape of data evolves over time. Fields get added, renamed, changed in type, split into two, or merged into one. Handling this safely requires a principled approach: transformations must be describable, executable, and where possible reversible.

ZIO Blocks Schema provides a powerful reflection layer via `DynamicValue` and `DynamicOptic`. The missing piece was a first-class migration API: a way to describe structural transformations between schema versions, execute them against `DynamicValue` at runtime, compose them into upgrade paths, and reverse them for downgrade scenarios. This contribution adds that layer.

---

## What Was Implemented

The implementation spans four primary files introduced across commits `9eac545`, `0abe1d1`, and `0fa2411`:

| File | Description |
|------|-------------|
| `DynamicMigration.scala` | Untyped, serializable migration core |
| `MigrationExpr.scala` | Serializable AST for value transformations |
| `TypedMigration.scala` | Type-safe migration with schema witnesses |
| `MigrationRegistry.scala` | Registry for multi-version migration chains |

---

## DynamicMigration

`DynamicMigration` is the foundational type: a `Chunk[MigrationAction]` that operates on `DynamicValue`. It is fully serializable — no closures, no runtime reflection. All transformations are encoded as data.

```scala
trait DynamicMigration {
  def apply(value: DynamicValue): Either[MigrationError, DynamicValue]
  def reverse: DynamicMigration
  def ++(other: DynamicMigration): DynamicMigration
}
```

- `apply` executes the migration as a left fold over the action list. Each action transforms the `DynamicValue` in sequence; the first failure short-circuits with `Left[MigrationError]`.
- `reverse` produces the structural inverse of the migration by reversing the action sequence and inverting each individual step. Every `MigrationAction` carries enough information to invert itself, so `reverse` is a pure structural operation with no external state.
- `++` composes two migrations sequentially. Composition is associative: `(a ++ b) ++ c == a ++ (b ++ c)`.

---

## MigrationAction ADT

`MigrationAction` is a sealed ADT of all supported migration operations. Each action is addressed via `DynamicOptic` (a path into the `DynamicValue` structure).

| Action | Description | Inverse |
|--------|-------------|---------|
| `AddField(path, expr)` | Insert a new field computed from a `MigrationExpr` | `DropField` (carries the same default for re-add) |
| `DropField(path, default)` | Remove a field; `default` is used when reversing | `AddField` |
| `RenameField(from, to)` | Move a field value from one path to another | `RenameField(to, from)` |
| `TransformValue(path, expr, inverseExpr)` | Apply a `MigrationExpr` to the value at a path | `TransformValue(path, inverseExpr, expr)` |
| `Optionalize(path)` | Wrap a field value in `Some` | `Mandate` |
| `Mandate(path)` | Unwrap `Some`; fails on `None` | `Optionalize` |
| `Join(pathA, pathB, dest, expr, splitExpr)` | Merge two fields into one via a combining expression | `Split` |
| `Split(src, pathA, pathB, exprA, exprB, mergeExpr)` | Decompose a field into two via projection expressions | `Join` |
| `RenameCase(from, to)` | Rename a variant case by label in a `DynamicValue.Variant` | `RenameCase(to, from)` |
| `ChangeFieldType(path, expr, inverseExpr)` | Apply a type-conversion expression with its inverse | `ChangeFieldType(path, inverseExpr, expr)` |

Every action carries its own inverse information. `DynamicMigration.reverse` is therefore O(n) in the number of actions and requires no external schema information.

---

## MigrationExpr

`MigrationExpr` is a serializable AST for value-to-value transformations. No user lambdas are admitted; all expressions are data structures that can be serialized, stored, and replayed.

```scala
sealed trait MigrationExpr

object MigrationExpr {
  case class Literal(value: DynamicValue)     extends MigrationExpr
  case object IntToString                      extends MigrationExpr
  case object StringToInt                      extends MigrationExpr
  case class StringConcat(separator: String)   extends MigrationExpr  // combines _left and _right fields
  case class StringSplit(separator: String)    extends MigrationExpr  // produces _left and _right
  case class Conditional(
    predicate: MigrationExpr,
    ifTrue: MigrationExpr,
    ifFalse: MigrationExpr
  )                                            extends MigrationExpr
  case class Compose(first: MigrationExpr,
                     second: MigrationExpr)    extends MigrationExpr
  case object Identity                         extends MigrationExpr
}
```

Expressions are evaluated by a small interpreter in `MigrationExpr.eval`. The interpreter is total — it returns `Either[MigrationError, DynamicValue]`, never throws.

---

## TypedMigration

`TypedMigration[A, B]` wraps a `DynamicMigration` with schema witnesses for the source type `A` and target type `B`. This provides:

- Compile-time evidence that the migration transforms `A` to `B`
- Automatic `DynamicValue` encoding/decoding via ZIO Blocks Schema
- Type-safe composition: `TypedMigration[A, B] ++ TypedMigration[B, C]` produces `TypedMigration[A, C]`
- Type-safe reversal: `TypedMigration[A, B].reverse` produces `TypedMigration[B, A]`

```scala
val migration: TypedMigration[UserV1, UserV2] =
  TypedMigration.from[UserV1, UserV2](
    DynamicMigration(
      MigrationAction.AddField(
        DynamicOptic.field("emailVerified"),
        MigrationExpr.Literal(DynamicValue.Primitive(false))
      ),
      MigrationAction.RenameField(
        DynamicOptic.field("name"),
        DynamicOptic.field("displayName")
      )
    )
  )
```

---

## MigrationRegistry

`MigrationRegistry` stores a directed chain of migrations between successive schema versions. Given a source version and a target version, it composes the intermediate steps automatically.

```scala
val registry = MigrationRegistry.empty
  .register(migration_v1_to_v2)   // TypedMigration[UserV1, UserV2]
  .register(migration_v2_to_v3)   // TypedMigration[UserV2, UserV3]

// Automatically composed: v1 → v2 → v3
val v1ToV3: Either[MigrationError, TypedMigration[UserV1, UserV3]] =
  registry.migrate[UserV1, UserV3]

// Automatically reversed: v3 → v2 → v1
val v3ToV1 = v1ToV3.map(_.reverse)
```

---

## Design Decisions

**Why `DynamicValue` instead of typed AST?**
`DynamicValue` is the universal runtime representation in ZIO Blocks Schema. Operating at this level means migrations are independent of the Scala type system at runtime — they can be serialized to JSON, stored in an event log, and replayed against any value that matches the schema, without recompiling or loading the original class.

**Why no lambdas in `MigrationExpr`?**
Lambdas cannot be serialized. Storing a migration containing a lambda in an event store or configuration database would require JVM class serialization, which is fragile across versions and impossible across languages. The `MigrationExpr` AST is designed to be serializable to any format ZIO Blocks Schema supports.

**Why carry inverse information on every action?**
Alternative: compute inverses from schema information at the point of reversal. This couples the migration system to schema availability and requires the target schema to be loadable at reversal time — not always true in a distributed system where old schema versions may no longer be on the classpath. Carrying inverse information on each action makes `reverse` self-contained.

---

## Usage Example

```scala
import zio.blocks.schema._
import zio.blocks.schema.migration._

// V1: name field, no emailVerified
case class UserV1(name: String, email: String)
object UserV1 {
  implicit val schema: Schema[UserV1] = DeriveSchema.gen
}

// V2: displayName instead of name, emailVerified added
case class UserV2(displayName: String, email: String, emailVerified: Boolean)
object UserV2 {
  implicit val schema: Schema[UserV2] = DeriveSchema.gen
}

val migration: TypedMigration[UserV1, UserV2] =
  TypedMigration.from[UserV1, UserV2](
    DynamicMigration(
      MigrationAction.RenameField(
        DynamicOptic.field("name"),
        DynamicOptic.field("displayName")
      ),
      MigrationAction.AddField(
        DynamicOptic.field("emailVerified"),
        MigrationExpr.Literal(DynamicValue.fromPrimitive(false))
      )
    )
  )

val v1User = UserV1("Alice", "alice@example.com")
val result: Either[MigrationError, UserV2] = migration(v1User)
// Right(UserV2("Alice", "alice@example.com", false))

val downgrade = migration.reverse
val back: Either[MigrationError, UserV1] = downgrade(result.toOption.get)
// Right(UserV1("Alice", "alice@example.com"))
```

---

## Running the Tests

```bash
sbt "schema/test"
# or run only migration tests
sbt "schema/testOnly *Migration*"
```

---

## Upstream

This fork tracks [zio/zio-blocks](https://github.com/zio/zio-blocks). The schema migration contribution is isolated to the `schema-migration` branch. The `main` branch contains only this documentation update.

---

## License

Apache License, Version 2.0 — see upstream [LICENSE](https://github.com/zio/zio-blocks/blob/main/LICENSE).

This is a fork and contribution to the ZIO Blocks project. All code contributed here is offered under the same Apache 2.0 license as the upstream project.
