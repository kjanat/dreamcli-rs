# Example CLIs

Each example defines its CLI in `app() -> Cli` and keeps supporting types in the
same file. `main()` calls `app().run()` or awaits `app().run_async()`. Tests import
the same application and run it with injected inputs.

| Example                          | Demonstrates                                                                        |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| [release.rs](release.rs)         | Version parsing, release plans, conditional credentials, and custom JSON encodings  |
| [batch.rs](batch.rs)             | File checks, collections, limits, collision detection, tables, and progress         |
| [settings.rs](settings.rs)       | Nested commands, global flags, configuration sources, redaction, and JSON Schema    |
| [terminal.rs](terminal.rs)       | Spinners, progress bars, status messages, failure cleanup, and JSON or quiet output |
| [async_build.rs](async_build.rs) | Typed inputs and state, async handlers, Tokio timers, and progress                  |
| [manifest.rs](manifest.rs)       | Named inputs, shared pipelines, reusable middleware, and concurrent file hashing    |
| [greet.rs](greet.rs)             | A positional argument, a boolean flag, and a repeat count                           |
| [deploy.rs](deploy.rs)           | Enum choices, global flags, credentials, derived state, and nested commands         |

The planners simulate work and return plans. The manifest example writes a file
selected by `--out`. The release and settings examples require `url`; Cargo lists
each example's required features. The commands below also use default features.

## Manifest fingerprinting

```sh
cargo run -p fujo --example manifest -- build . --out manifest.json
cargo run -p fujo --example manifest -- verify . --manifest manifest.json
cargo run -p fujo --example manifest -- verify . --manifest manifest.json --json
cargo run -p fujo --example manifest -- build . --out manifest.json --force --jobs 8 --ignore .git,target,docs
cargo run -p fujo --example manifest -- verify . --explain
```

`build` walks a directory, skips names in `--ignore`, and hashes files with
`--jobs` concurrent workers. It writes relative paths, sizes, and FNV-1a digests
to JSON. `verify` repeats the scan and reports changed, missing, and added files
as a table or JSON. Differences produce exit code 1. `--force` permits replacing
an existing manifest.

A shared `flags!` struct declares `--jobs` and `--ignore`. Each command includes
it in its `inputs!` struct and reads fields such as `globals.jobs`. A root
pipeline creates the worker semaphore once per invocation through `Cli::share`.

Both commands inherit a scan pipeline. It reads the semaphore, lists files in a
synchronous derivation, and uses `Timed` middleware to measure later stages.
Each command then hashes files asynchronously under the semaphore and reports
progress. `build` checks the output path with `tap` before scanning. `verify`
prints a summary through middleware before its action renders the table.
`--explain` lists these stages without running them.

Environment bindings are `MANIFEST_JOBS` and `MANIFEST_IGNORE`. Configuration
keys are `jobs`, `ignore`, and `manifest`.

## Typed async build

```sh
cargo run -p fujo --example async_build
cargo run -p fujo --example async_build -- --profile=prod --jobs=20 --delay=150ms
cargo run -p fujo --example async_build -- --delay=0ms --jobs=2 --json
```

The handler receives a shared profile, a nonzero job count, and a `Duration`.
An async derivation creates `Build` state for the action. `tokio::time::sleep`
waits between progress updates while a spinner animates. Tokio belongs to the
example; Fujo uses the application's executor.

`greet` uses tuple inputs and `manifest` uses named structs. The release, batch,
settings, and terminal examples use dynamic handles. Those handles check value
types at compile time and registration or state availability at runtime. Typed
callbacks also check their input bundle and state types at compile time.

## Terminal output

Run this in a terminal:

```sh
cargo run -p fujo --example terminal
```

The demo takes about five seconds. It shows a spinner while discovering inputs,
changes its label for dependency checks, advances a progress bar for 24 items,
then verifies the result and prints a table. A status message appears halfway
through. The spinner's timer draws frames every 80 ms, including while the
action sleeps.

```sh
cargo run -p fujo --example terminal -- --steps=40 --delay=150ms --verbose
cargo run -p fujo --example terminal -- --fail-at=8
cargo run -p fujo --example terminal -- --delay=0ms --json
cargo run -p fujo --example terminal -- --quiet
```

`--steps` accepts a `NonZeroU16` from 1 to 100. `--delay` accepts a `Duration` up
to one second per item. `--fail-at` must name an existing item. A simulated
failure clears the bar, reports `DEMO_FAILURE`, exits with code 2, and skips the
result table.

`--verbose` adds debug messages. `--quiet` suppresses debug, status, and animation.
JSON mode writes a result array to stdout and human messages to stderr. Piped
output suppresses animation.

Release and batch plans finish too soon to show several animation frames.
This demo adds delays so the animation can be inspected. Capture stores activity
events without frames. `scripts/check-terminal.py` checks timing and rendering
through a Unix PTY.

## Release planner

Pass an existing artifact, such as the workspace manifest:

```sh
cargo run -p fujo --features url --example release -- plan Cargo.toml --from=1.4.2 --bump=minor --json
```

The result contains `current: "1.4.2"` and `next: "1.5.0"`. `Version` and `Bump`
are Rust input types without Serde implementations. The resulting plan has its
own serializable type.

`--publish` adds an authenticated publication step to the plan. Supply a token
through `SHIPYARD_TOKEN`, `--token`, or the terminal prompt. Missing credentials
in a noninteractive run produce `AUTH_REQUIRED`. The plan excludes token values.

The example searches for `shipyard` configuration:

```json
{
  "release": {
    "from": "1.4.2",
    "bump": "minor",
    "registry": "https://registry.example.invalid"
  }
}
```

`SHIPYARD_VERSION` overrides `release.from`, and `--from` overrides both. Registry
URLs require HTTPS. Versions contain three decimal `u32` components without
leading zeros. Increment operations check for overflow.

## Batch planner

```sh
cargo run -p fujo --example batch -- plan Cargo.toml README.md \
  --output planned-output --workers 8 --timeout 1m30s --budget 2mb \
  --tag fast,local,fast --priority README.md=20 --sources --json
```

This produces one job per input with unique tags, numeric priorities, eight
workers, a 90,000 ms timeout, and a 2,097,152-byte budget. `--sources` adds each
value's source. Omit `--json` to print a table.

Workers use `NonZeroU16` from 1 to 64. Priorities use `u16` from 0 to 100, and
priority keys must be unique. Input files must exist; the output directory can
be new. Duplicate target names produce `TARGET_COLLISION`. Collision detection
compares paths lexically rather than resolving symlinks or filesystem aliases.

Environment bindings are `BATCH_LABEL`, `BATCH_WORKERS`, and `BATCH_TAGS`.
The `batch` configuration accepts `mode` and `workers`.

## Nested settings inspector

```sh
cargo run -p fujo --features url --example settings -- config show --profile prod --port 9443 --json
cargo run -p fujo --features url --example settings -- config validate --endpoint http://localhost
cargo run -p fujo --features url --example settings -- config schema
printf '%s\n' 'example-token' | cargo run -p fujo --features url --example settings -- config show --token=- --json
```

The first command reports resolved values and their sources. The second accepts
HTTP under the default development profile. Adding `--profile prod` rejects it
with `HTTPS_REQUIRED`. The schema command includes inherited global inputs.

`--token=-` reads stdin once. Tables and JSON show `[redacted]`; derived state
can inspect whether credentials are present. Environment bindings are
`SETTINGS_PROFILE`, `SETTINGS_PORT`, `SETTINGS_ENDPOINT`, `SETTINGS_FEATURES`,
and `SETTINGS_TOKEN`.

Example `settings` configuration:

```json
{
  "profile": "prod",
  "server": { "port": 9443, "endpoint": "https://service.example.invalid" },
  "features": ["cache", "metrics"],
  "limits": { "requests": 50 }
}
```

`--config PATH` selects a file. Otherwise the examples search project, user,
and system directories as described in the main README.

## Greeting

```sh
cargo run -p fujo --example greet -- Ada --loud --times 2
cargo run -p fujo --example greet -- --help
cargo run -p fujo --example greet -- completions bash
```

`greet` takes a name, `--loud` or `-l`, and a `u16` repeat count through `--times`
or `-t`. Its action prints the greeting that many times.

## Deployment planner

```sh
cargo run -p fujo --example deploy -- db seed --json
DEPLOY_TOKEN=example-token cargo run -p fujo --example deploy -- deploy prod --region eu --json
```

`deploy` requires `--token` or `DEPLOY_TOKEN`. Missing credentials produce
`AUTH_REQUIRED` during derivation. The global `--region` flag parses a Rust enum
with `us` and `eu` choices. It also reads `DEPLOY_REGION` or `deploy.region`.
`db seed` runs without credentials. Both commands simulate deployment.

## Type and behavior checks

```sh
cargo test -p fujo --features url --test example_clis
cargo test -p fujo --test typed --test async_execution --test lifecycle_reference
cargo test -p fujo --doc
cargo test -p fujo-core --doc
cargo check -p fujo --features url --examples
```

Handlers annotate inputs with types such as `Version`, `Bump`, `NonZeroU16`,
`PathBuf`, `Duration`, `Vec<PathBuf>`, and `BTreeMap<String, u16>`. Compile-fail
doctests cover wrong value types, collection types, defaults, input kinds, and
source bindings on element parsers.

Runtime tests cover source precedence, credentials and redaction, prompt answers,
invalid versions, range checks, files, collisions, collections, progress, nested
commands, schemas, JSON help, and all four completion generators. They check that
planners leave input files unchanged and create no output directory.

Typed-command doctests cover undeclared inputs, mismatched bundles, missing
state, positional globals, invalid sync and async combinations, and reused
continuations. Runtime tests cover borrowed non-Clone inputs, local async state,
global flags, derivation failures, middleware order and early returns, reusable
stages, retry and timeout behavior, inherited and shared pipelines, stage
inspection, named bundles, and dropped invocations.

Manifest tests use temporary directories to check digests, differences,
overwrite protection, and CLI, environment, and configuration precedence.

All examples provide help, JSON help, and Bash, Zsh, Fish, and PowerShell
completions:

```sh
cargo run -p fujo --example batch -- plan --help
cargo run -p fujo --features url --example release -- plan --help --json
cargo run -p fujo --features url --example settings -- completions fish
```
