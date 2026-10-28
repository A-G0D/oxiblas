# AGENTS.md - OxiBLAS

OxiBLAS is a pure-Rust BLAS/LAPACK implementation for the SciRS2 ecosystem,
published as a Cargo workspace under `crates/`. It is a library, not an
application: default features must stay pure Rust with no C/C++/Fortran
dependencies.

Rust 1.85+ (MSRV). Edition 2024. License Apache-2.0. Default branch `master`.

## Read first

- `CONTRIBUTING.md` - workspace layout, code-quality requirements, testing, and
  commit/PR process; authoritative for contributions.
- `README.md` - feature and crate overview, supported types, performance notes.
- `SECURITY.md` - report vulnerabilities privately; do not open a public issue
  or PR for an unfixed security problem.
- `CHANGELOG.md` - add user-visible changes under `[Unreleased]`.
- `deny.toml`, `clippy.toml`, `rustfmt.toml` - dependency bans, clippy MSRV,
  and formatting edition.

If this file conflicts with `CONTRIBUTING.md`, `CONTRIBUTING.md` wins.

## Workspace layout

| Path | Contents |
|---|---|
| `crates/oxiblas-core/` | Scalar types, SIMD abstraction, memory primitives; `no_std` + alloc capable. |
| `crates/oxiblas-matrix/` | Dense/packed/banded/memory-mapped matrix storage. |
| `crates/oxiblas-blas/` | BLAS levels 1-3 and the CBLAS C-ABI surface. |
| `crates/oxiblas-lapack/` | LAPACK-equivalent factorizations and solvers. |
| `crates/oxiblas-sparse/` | Sparse formats, solvers, and preconditioners. |
| `crates/oxiblas-ndarray/` | Optional `ndarray` interoperability. |
| `crates/oxiblas/` | Facade crate re-exporting the publishable crates. |
| `crates/oxiblas-benchmarks/` | `publish = false` benchmark-only crate; excluded from normal commands. |
| `crates/oxiblas-ffi/` | Retired; excluded from the workspace and never built. |
| `fuzz/` | Independent `[workspace]` with libFuzzer targets; nightly only. |

`fuzz/` is deliberately not a root workspace member (see `Cargo.toml` and
`fuzz/Cargo.toml`). Do not add it to `members` or make root commands require
`libfuzzer-sys`. The GitHub Actions workflows are checked in with `.disabled`
suffixes and do not run automatically, so run the checks locally.

## Build / test / lint (exact commands)

Run from the repository root.

| Task | Command |
|---|---|
| Build workspace | `cargo build --workspace --exclude oxiblas-benchmarks` |
| Build one crate | `cargo build -p oxiblas-blas` |
| Test workspace | `cargo test --workspace --exclude oxiblas-benchmarks` |
| Test one crate | `cargo test -p oxiblas-core` |
| One test | `cargo test -p oxiblas-core <test-name>` |
| Doctests | `cargo test --doc --workspace --exclude oxiblas-benchmarks` |
| `no_std` core | `cargo test -p oxiblas-core --no-default-features` |
| Lint (must be clean) | `cargo clippy --workspace --exclude oxiblas-benchmarks --all-targets -- -D warnings` |
| Format check | `cargo fmt --all -- --check` (fix with `cargo fmt --all`) |
| Docs | `RUSTDOCFLAGS="-D warnings" cargo doc --workspace --exclude oxiblas-benchmarks --no-deps` |
| Dependency bans | `cargo deny check bans` (requires `cargo-deny`) |
| Fuzz target | `cd fuzz && cargo +nightly fuzz run mtx_matrix_market` (or `mmap_header`; requires nightly + `cargo-fuzz`) |

`cargo nextest run --workspace --exclude oxiblas-benchmarks` is an accepted
alternative to `cargo test` when nextest is installed.

Avoid `--all-features` in the normal loop (`CONTRIBUTING.md`): the optional
`compare-openblas` benchmark comparison needs a system OpenBLAS install and is
not part of the normal development loop. Run only the commands above unless the
task explicitly concerns that comparison.

## Testing expectations

- New public behavior needs a unit test; bug fixes need a regression test that
  fails without the fix (`CONTRIBUTING.md`).
- Tests and doctests are exempt from the no-`unwrap()` rule; production code is
  not.
- The checked-in CI definition (`.github/workflows/ci.yml.disabled`) has a
  dedicated `no_std` check: `cargo test -p oxiblas-core --no-default-features`.
  Run it when touching `oxiblas-core` or shared matrix primitives.
- Do not weaken, skip, or delete tests or doctests to make a change pass. If a
  test is wrong, argue that separately.

## Code conventions

- No warnings: clippy runs with `-D warnings`; do not add `#[allow(...)]` to
  silence a real finding.
- No `unwrap()`/`expect()` in production code; return `Result`/`Option` or use a
  documented, checked invariant.
- Every `unsafe fn` needs a `# Safety` doc section stating its preconditions;
  every `unsafe { ... }` block needs a `// SAFETY:` comment explaining why the
  preconditions hold at that call site.
- Keep source files under 2000 lines; split modules and re-export when a file
  grows past that (`CONTRIBUTING.md`).
- Add new dependencies to `[workspace.dependencies]` in the root `Cargo.toml`
  and reference them with `workspace = true`; do not pin versions in member
  manifests.
- Default features must not pull in C/C++/Fortran dependencies. Any FFI belongs
  behind a non-default feature.
- `Cargo.toml`'s `rust-version` and `clippy.toml`'s `msrv` must stay in sync
  (currently 1.85).
- Respect the banned-crate list in `deny.toml`; for example COOLJAPAN
  replacements for `bincode`, `rustfft`, `zip`/`flate2`/`zstd`, `rusqlite`,
  `z3`, and the OpenBLAS family.
- Use `snake_case` and standard Rust naming conventions.

## PR and commit rules

- For anything beyond a small fix, open an issue first to agree on the approach
  (`CONTRIBUTING.md`).
- Keep commits focused; describe the *why*, not just the *what*.
- Update `CHANGELOG.md` under `[Unreleased]` for user-visible changes. Do not
  bump crate versions; maintainers handle releases.
- Security problems go through `SECURITY.md`, not a public issue or PR.
- Do not rename or enable the `.github/workflows/*.yml.disabled` files as part
  of an unrelated change.
- AI-assisted changes: disclose the assistance in the PR description. The human
  submitter reviews the diff and is accountable for it.
- Contributions are licensed under Apache-2.0.

## Do not touch

- `crates/oxiblas-ffi/` - retired, excluded from the workspace, never built.
- `crates/oxiblas-benchmarks/` - `publish = false`; do not move it into the
  normal build/test path or add default dependencies there.
- `fuzz/` - separate workspace; do not merge it into the root workspace.
- `Cargo.lock` - gitignored for this library workspace; do not commit it.
- `deny.toml` bans, `clippy.toml` MSRV, and `rustfmt.toml` edition - change only
  when that policy change is the task.

## Definition of done

- [ ] The change is scoped and linked to an issue, or is a small self-contained
      fix.
- [ ] `cargo fmt --all -- --check`, clippy with `-D warnings`, and the relevant
      tests/doctests pass.
- [ ] New behavior has a test; the `no_std` core check passes if core changed.
- [ ] `CHANGELOG.md` updated for user-visible changes; no version bump.
- [ ] No new C/C++/Fortran dependency in default features, and every new
      dependency is in `[workspace.dependencies]`.
- [ ] No secrets, credentials, or unrelated reformatting in the diff.
- [ ] AI-assistance disclosure is present in the PR description.

## When something fails

1. Read the actual compiler/test output; reproduce the failure before changing
   code.
2. Check `CONTRIBUTING.md` for the project-specific rule that applies.
3. Do not mask failures with `#[allow]`, ignored tests, feature changes, or
   version pins.
4. If a failure predates your change, say so and keep it separate from your
   diff.
