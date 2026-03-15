# dependabot-cargo-workspace-sample

Reproduction repository for a Dependabot Cargo workspace issue where `pin_version` does not apply to `[workspace.dependencies]`, causing a version mismatch error.

## The Problem

When Dependabot updates a Cargo dependency in a workspace, its `LockfileUpdater` calls `pin_version` to set an exact version constraint (e.g., `=4.11.0`). However, `pin_version` only processes `[dependencies]`, `[dev-dependencies]`, and `[build-dependencies]` — it does not process `[workspace.dependencies]`.

In a workspace member with `actix-web.workspace = true`, `pin_version` adds `version = "=4.11.0"` to the member's `[dependencies]` entry. But Cargo ignores the `version` key when `workspace = true` is present:

```
warning: unused manifest key: dependencies.actix-web.version
```

The actual version constraint comes from `[workspace.dependencies]`, which the `ManifestUpdater` changed to `"4.11.0"` (interpreted as `^4.11.0`). Cargo resolves to the latest version matching `^4.11.0` (e.g., 4.13.0), but Dependabot expects exactly 4.11.0. The mismatch triggers `RuntimeError: Failed to update actix-web!`, which surfaces as `unknown_error: null` in GitHub Actions.

## Reproduction Steps

### Prerequisites

- Rust toolchain installed
- This repository cloned with its `Cargo.lock` (contains actix-web 4.9.0, actix-server 2.5.0)

### Step 1: Verify initial state

```console
$ grep -A1 '^name = "actix-web"$' Cargo.lock
name = "actix-web"
version = "4.9.0"

$ grep -A1 '^name = "actix-server"$' Cargo.lock
name = "actix-server"
version = "2.5.0"
```

### Step 2: Simulate Dependabot's ManifestUpdater

Change `[workspace.dependencies]` in `Cargo.toml`:

```diff
-actix-web = "4.4"
+actix-web = "4.11.0"
```

### Step 3: Run cargo update (as Dependabot's LockfileUpdater does)

```console
$ cargo update -p actix-web:4.9.0
    Updating actix-server v2.5.0 -> v2.6.0
    Updating actix-web v4.9.0 -> v4.13.0
```

### Step 4: Observe the mismatch

Cargo selected **4.13.0** (latest matching `^4.11.0`), but Dependabot's `desired_lockfile_content` expects **4.11.0**.

```ruby
# In lockfile_updater.rb (before PR #12487 fix):
desired = 'name = "actix-web"\nversion = "4.11.0"'
lockfile.include?(desired)  # => false (lockfile has 4.13.0)
raise "Failed to update actix-web!"  # => unknown_error: null in GitHub Actions
```

## Root Cause

`pin_version` in [`lockfile_updater.rb`](https://github.com/dependabot/dependabot-core/blob/main/cargo/lib/dependabot/cargo/file_updater/lockfile_updater.rb) iterates over `DEPENDENCY_TYPES = %w(dependencies dev-dependencies build-dependencies)` but does not process `workspace.dependencies`. The exact version constraint `=4.11.0` is never applied to the workspace dependency declaration.

## Fix

[PR #12487](https://github.com/dependabot/dependabot-core/pull/12487) added a fallback that accepts any version newer than the previous version, working around this issue. The root cause (`pin_version` not handling `[workspace.dependencies]`) remains unfixed.

## Related

- [dependabot/dependabot-core#12487](https://github.com/dependabot/dependabot-core/pull/12487) — The workaround fix
- [uuushiro/dependabot-cargo-sample](https://github.com/uuushiro/dependabot-cargo-sample) — Reproduction for a related but different issue (direct dependency causing intermediate version stop)
