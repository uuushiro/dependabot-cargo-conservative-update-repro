# dependabot-cargo-workspace-sample

Minimal reproduction for `cargo update -p` conservative update behavior that caused Dependabot `unknown_error: null`.

## The Problem

`cargo update -p actix-web` stops at version 4.10.2 instead of reaching 4.13.0, because the conservative update strategy does not update transitive dependencies (actix-server, once_cell) that are already satisfied by the intermediate version.

```console
$ cargo update -p actix-web --verbose
   Unchanged actix-server v2.5.0 (available: v2.6.0)
    Updating actix-web v4.9.0 -> v4.10.2 (available: v4.13.0)
   Unchanged once_cell v1.20.1 (available: v1.21.4)
```

actix-web 4.10.2 requires `actix-server ^2` and `once_cell ^1.5`, both satisfied by the locked versions. actix-web 4.11.0+ requires `actix-server ^2.6` and `once_cell ^1.21`, which the locked versions cannot satisfy.

## Reproduction

```console
$ git clone https://github.com/uuushiro/dependabot-cargo-workspace-sample
$ cd dependabot-cargo-workspace-sample
$ cargo update -p actix-web
    Updating actix-web v4.9.0 -> v4.10.2 (available: v4.13.0)
```

### Why sentry is needed

Without `sentry` + `sentry-actix`, Cargo's resolver updates actix-server and once_cell to reach actix-web 4.13.0. The additional complexity from sentry's dependency graph causes the resolver to prefer falling back to actix-web 4.10.2 rather than updating the locked transitive dependencies.

## Cargo's documentation

From [`cargo update`](https://doc.rust-lang.org/cargo/commands/cargo-update.html):

> Its transitive dependencies will be updated only if SPEC cannot be updated without updating dependencies.

Cargo's test suite documents this behavior in [`transitive_minor_update`](https://github.com/rust-lang/cargo/blob/master/tests/testsuite/update.rs#L65) with the comment:

> Also note that this is probably **counterintuitive and weird**. We may wish to change this one day.

## Fix

The solution is to update the transitive dependencies together:

```console
$ cargo update -p actix-web -p actix-server -p once_cell
    Updating actix-server v2.5.0 -> v2.6.0
    Updating actix-web v4.9.0 -> v4.13.0
    Updating once_cell v1.20.1 -> v1.21.4
```

Or update everything:

```console
$ cargo update
```

## Related

- [dependabot-core PR #12487](https://github.com/dependabot/dependabot-core/pull/12487) — Dependabot fix to accept partial updates
- [Cargo `transitive_minor_update` test](https://github.com/rust-lang/cargo/blob/master/tests/testsuite/update.rs#L65) — Cargo's own test for this behavior
