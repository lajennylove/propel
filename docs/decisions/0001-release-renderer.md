# 0001: How release builds render views until the compiler exists

Status: **Accepted: option B.** Release builds are refused until `propel_compiler` exists. Developer steps are in [`docs/production.md`](../production.md).

## Decision

Option B. `propel_rocket` has a `compile_error!` under `cfg(not(debug_assertions))`. No production release is possible before the compiler lands. Option C is not implemented and is kept here for reference.

## Context

The plan says release builds render compiled views, with typed contexts and no runtime template lookup. The dev renderer (`propel_runtime`) is an interpreter. It re-reads `.propel` files and re-parses them on every request, and it renders from a `serde_json::Value` context.

`propel_compiler` is not built, so release builds currently use the same interpreter as dev builds. The code has no `cfg(debug_assertions)` gate. The README discloses this, but nothing in the code stops it.

What release does today, and what the plan says it should do:

| Concern | Plan (release) | Today (release) |
| --- | --- | --- |
| How views are rendered | compiled Rust | interpreted from `.propel` text |
| Template source | compiled into the binary | read from `views/` on each request |
| Context | typed structs | `serde_json::Value` |
| Missing variable | compile error | runtime error (500) |
| Parsing per request | none | yes, about 23 µs for `posts.index` and `layouts.app` |
| Deployment | binary only | binary **and** a `views/` directory on disk |

Two things make this worse than a missing feature:

- **The deployed server reads files from disk.** If the working directory or `views/` is wrong in production, every page fails at runtime. There is no check at startup.
- **Errors are caught at the wrong time.** A view that names a missing field only fails when a request reaches it, not at build or startup.

Nothing has been deployed. So the cost of each option today is low, but it grows as soon as a Propel app goes to production.

## Options

### A. Keep the interpreter in release (status quo)

- **Pros:** works now; one code path, so dev and release behave the same.
- **Cons:** breaks the plan's release model, and keeps disk reads, re-parsing and `Value` contexts in production. Deployments need `views/` on disk.

### B. Refuse to build release until the compiler exists

Add a `compile_error!` under `cfg(not(debug_assertions))`, or a Cargo feature that is off by default for release.

- **Pros:** the release model cannot drift. Nothing reaches production without the compiler.
- **Cons:** no release build of any Propel app until `propel_compiler` lands. This is fine if nothing is planned for production before then.

### C. Interim release mode: embed and cache

Release reads templates at compile time (`include_str!`, or a build step that embeds the `views/` directory) and parses each template once per process, caching the AST. Rendering still uses the `Value` context.

- **Pros:** no disk reads or re-parsing in production, and `views/` no longer has to be deployed. Releases keep working. The change is small and the dev path is untouched.
- **Cons:** it's a second interim design that will be replaced by the compiler. The `Value` context and runtime "undefined variable" errors remain.

## Recommendation

**Choose B if no Propel app is planned for production before `propel_compiler` lands.** It is the strictest option and costs nothing today.

**Choose C if production comes first.** It removes the disk and re-parse costs without waiting for the compiler, and it's the smallest step toward the plan.

Option A should not be used for production.

## Consequences of any choice

- The `README.md` "Release builds" section must match whichever option is picked.
- The benchmark in `propel_runtime/benches/render.rs` is the baseline for C. A cache should show a clear drop in the "render" numbers, which include the disk read.
- Whichever option is picked, the dev renderer stays debug-only in intent. Release never falls back to reading `views/` from the working directory without the owner's sign-off.

## Questions for the owner

1. Is a Propel app planned for production before `propel_compiler` is built?
2. If yes, is interim option C acceptable, given it keeps the `Value` context until the compiler lands?
