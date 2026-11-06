# Going to production with Propel

**Short answer: you cannot release a Propel app to production yet.** `cargo build --release` fails by design, with a compile error from `propel_rocket`. This is decision [0001](decisions/0001-release-renderer.md), option B.

This page is for developers using Propel in their own Rocket projects. It covers what works today, what does not, and the steps needed to ship once the compiler exists.

## Why release is blocked

Development and production use different renderers:

| | Dev build (`cargo run`) | Release build (`cargo build --release`) |
| --- | --- | --- |
| Views | read from `views/` on every request | would also read from `views/` (not allowed) |
| Context | `serde` values, checked at runtime | would need typed structs, checked at compile time |
| Template errors | shown on the error page | would be hidden (generic 500) |

Shipping the dev renderer would mean a production server that reads template files from disk on every request, that needs a `views/` folder next to the binary, and that finds missing variables only when a user hits the page. The compiled path removes all three problems. Until it exists, the release build is refused.

## What you can do today

- Build, run and iterate in dev mode, as described in [`examples/propel/README.md`](../../examples/propel/README.md).
- Write typed context structs for every view now. They are what the compiler will check.
- Keep each view's context complete. A missing field fails at render time today, so the dev server and the tests are where you find it.
- Run `cargo check` and `cargo test` regularly.

## Steps to production (once `propel_compiler` lands)

Each step is required. Check them off in order.

1. **Upgrade Propel.** Bump the `rev` in `Cargo.toml` to a commit that includes `propel_compiler`. Run `cargo update -p propel_rocket`.
2. **Make every view's context a typed struct.** One struct per view. Derive `Serialize` only if you still need it for development. Handlers must fill every field.
3. **Compile the views.** Add the compiled-view macro (name to be decided) for each view, or the build step if the compiler uses one. Views are compiled into the binary.
4. **Make `cargo check` clean.** Every template error must be fixed at this point. A release build must not depend on runtime checks.
5. **Run the shared tests.** Each view needs a golden test: a typed context, the expected HTML, and an exact comparison. The same cases must pass in dev and release.
6. **Confirm the binary does not read `views/`.** Run the release build from an empty directory. Every page must still render.
7. **Set the Rocket profile.** Run with `ROCKET_PROFILE=release`. Set `ROCKET_ADDRESS` and `ROCKET_PORT` for your host. Do not bind `0.0.0.0` on a machine without a firewall.
8. **Deploy the binary only.** No `views/` folder is needed on the server.
9. **Smoke-test every route** against the deployed binary.

## What production will not show

- Render errors return a generic `Internal Server Error`. The detail goes to the log. Check the log when a page fails.
- `@dump` and `@dd` output. Those are development tools and are not rendered in release builds. Remove them or leave them in; they add nothing to production.

## Security notes that apply in every build

- **URLs are checked.** An echo in `href`, `src`, `action`, `formaction` or `xlink:href` is replaced with `#` unless its scheme is `http`, `https`, `mailto` or `tel`, or it is relative. This applies to raw output too.
- **Everything else is HTML-escaped text.** `{{ }}` escapes `& < > " '`. That is not safe inside `<script>`, `<style>`, or unquoted JavaScript or CSS. Don't put echoes there. Use `@{{` for a literal `{{` in JavaScript or CSS.
- **Raw output is explicit.** `{!! !!}` is the only unescaped path. Do not use it with user-controlled data.
- **Template names are validated.** Names may contain only letters, digits, `_` and dots between them.

## Not allowed

- Removing or commenting out the `compile_error!` in `propel_rocket/src/lib.rs`. It is the gate. If you need to ship before the compiler exists, raise it with the Propel owner first. Option C in [0001](decisions/0001-release-renderer.md) is the alternative under discussion.
- Shipping a binary that was built with `--release` against a local copy of Propel with the gate removed.
