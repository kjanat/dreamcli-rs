# Port contract

Fujo ports [DreamCLI](https://github.com/kjanat/dreamcli) to Rust. The initial
reference was commit
[`f87c1672e10b27c066524a0715bfea0f714dd5bc`](https://github.com/kjanat/dreamcli/commit/f87c1672e10b27c066524a0715bfea0f714dd5bc)
in `../dreamcli`. Fujo uses Rust types and explicit host interfaces across five
crates.

## Preserved behavior

- Flags and arguments resolve from CLI, stdin, environment, configuration,
  prompts, then defaults.
- Source records distinguish explicit stdin selection from fallback stdin.
- Parsing supports aliases, inline values, short clusters, boolean negation,
  duplicate policies, counts, repeated arrays, key/value records, and case rules.
- Validation covers values, collection elements, splitting, stdin insertion,
  and declared defaults.
- Commands support required and optional inputs, variadic arguments, nesting,
  defaults, flag propagation, middleware, conditional prompts, and derived state.
- Builtins provide help and version priority, JSON and quiet modes, separators,
  structured errors, configuration, completions, tables, and terminal activities.
- In-process execution accepts injected environment variables, stdin, prompt
  answers, filesystem access, and output writers.

Reference tests compare 94 parsing and resolution cases and 11 lifecycle cases
with the recorded DreamCLI revision. CI checks the TypeScript generators before
running the Rust comparisons. Rust integration tests also cover plugin failures,
suspension, cancellation, and host operations. The fixtures define the tested
compatibility scope; text layout can differ between implementations.

## Rust API choices

| TypeScript API                          | Rust API                                                             |
| --------------------------------------- | -------------------------------------------------------------------- |
| Inferred object keys                    | Tuples, `inputs!` structs, or handles accessed with `ctx.get()`      |
| `undefined`                             | `Option<T>`                                                          |
| Required/defaulted inference            | Presence marker types; `get` returns `T` or `Option<T>`              |
| `number`                                | `f64` for compatibility, or any native integer through `parse::<T>`  |
| String paths                            | `PathBuf`                                                            |
| Millisecond numbers                     | `std::time::Duration`                                                |
| Byte size numbers                       | `u64`, rounded and range checked                                     |
| JavaScript `URL` / `Date`               | `url::Url` / `chrono::DateTime<Utc>`                                 |
| Standard Schema / custom callbacks      | `FromStr`, native decoder closures, and typed validators             |
| Context object intersections            | Typed owned state transformations, or a dynamic type-indexed store   |
| Runtime-specific Node/Bun/Deno adapters | `Runtime`, `FileSystem`, `ConfigReader`, `PromptEngine`, and `Write` |
| `package.json` discovery                | Explicit metadata or `cargo_manifest!()`                             |
| Process exit calls                      | Return `std::process::ExitCode` from `main`                          |

Applications provide an executor for async callbacks. See the
[async execution rules](README.md#async-execution). Configuration, prompts,
filesystem preparation, and stdio use synchronous operations. TTY spinners use a
standard-library timer thread, which stops and joins on completion or drop.
Terminal prompts read lines; secret prompts use a password reader.

`Command::inputs` connects declarations to borrowed handler parameters and owned
state. The callback receives a `Scope` with output and metadata. Derivations
replace the previous state; use tuples or structs to retain earlier values.
`inputs!` and `flags!` declare named bundles with the same type checks.

Typed middleware receives state and a continuation for later stages. Synchronous
middleware permits only synchronous later stages. `Middleware` and `Derive`
implementations can be reused across commands. Pipelines can be inherited by
commands or run once at the root. `--explain` lists the stages.

The dynamic API checks input and state access at runtime. `ctx.get()` looks up
an input by kind and name, then checks its Rust type. `ctx.state()` looks up state
by type; inserting another value of that type replaces the previous one.

Rust's activity guard records a failed finish when dropped after an action error.
DreamCLI capture omits that cleanup event. The lifecycle test checks this
difference. DreamCLI's unanswered `PROMPT_CANCEL` sentinel maps to `Answer::Skip`,
which permits a default. `Answer::Cancel` aborts with exit code 130.

`array()` defaults to an empty vector. `.optional()` removes that default and
represents absence as `None`. DreamCLI supplies empty collections even for
optional collection declarations.

Collections use `ValueParser<T>` elements from `value::string()`,
`value::parse::<T>()`, and related factories. Element parsers hold constraints
and encodings. Source bindings and defaults belong to the input. `InputSchema`
exposes declarations for reading and keeps codecs private. Collection methods
are available only on suitable types.

Framework diagnostics redact sensitive values. Handlers receive full values.
`Resolved::json()` redacts sensitive inputs and omits custom values without JSON
encodings. Common `FromStr` primitives have built-in encodings and schema
metadata. Other types can provide `encode_json` and `json_schema`. Help default
descriptions can differ from the runtime defaults.

Process-free runs need injected filesystem access. See the
[filesystem policies](README.md#configuration-and-testing). Filesystem interfaces
use `&Path`, including non-Unicode paths and custom native defaults. Converting a
positional input with `.variadic()` wraps its scalar default in a one-element
vector and preserves required-value access.

## Optional integrations

Cargo features control `regex`, `url`, and `chrono`. DreamCLI includes these
integrations by default. See [Cargo features](README.md#cargo-features).
`scripts/check-features.py` verifies behavior and dependencies for each build.

## Schema documents and formats

Rust definitions use `format: "fujo"` and `version: 1`. This format describes
Rust schema metadata rather than serializing handlers or reconstructing custom
Rust types. It differs from DreamCLI's definition format. Input schemas use
JSON Schema draft 2020-12 for values with JSON representations.

CLI arguments must be valid UTF-8. Invalid OS arguments produce a structured
parse error. String patterns use Rust regex syntax. Native help and terminal
renderers may differ in spacing, wrapping, and decoration. Compatibility tests
compare error categories, stable codes, exit statuses, and resolved values.
