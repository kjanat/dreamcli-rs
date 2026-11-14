# Optional integrations

Status: Implemented.

## Decision

Keep the five crates and the existing resolution order. The public crate enables
`completion`, `terminal-prompts`, and `terminal-activity` by default. `regex`,
`url`, and `chrono` are optional. Core has no default features; output enables
its terminal renderer by default. Internal dependencies disable defaults and
forward features explicitly. Features add capabilities under Cargo's feature
unification rules.

Gate each integration's factories, constraints, value kinds, and dependencies
together. A build must support every constraint its API allows applications to
declare. Duration and byte-size parsers use decimal scanners. Chrono uses a
separate ISO shape check. Neither needs `regex`.

Put terminal prompt and activity implementations in feature-gated modules.
Keep `PromptEngine`, scripted answers, activity handles, events, and cleanup
available in all builds. Without `terminal-activity`, handles record events but
start no timer and draw no output, including final activity messages. Logs and
tables still work. Without `terminal-prompts`, applications can still inject a
prompt engine.

Serde and JSON remain required for configuration, prompt values, codecs, and
metadata. Schema exports remain available. Keep resolution stages together until
a consumer needs separate feature controls or a metadata-only crate. Output can
be used without the parser. Completion uses core without optional integrations.

## Verification

The feature checker builds minimal, default, all-feature, and six individual
feature configurations. It tests behavior and checks normal dependencies across
all targets. It also tests minimal output and completion builds separately.
Workspace CI covers tests, Clippy, docs, DreamCLI fixtures, completion shells,
and terminal animation through a PTY.

## Consequences

Applications must enable `regex`, `url`, or `chrono` to use their factories and
constraints. Cargo metadata declares `url` as a requirement for the release and
settings examples. Another dependency can enable features an application has
disabled locally, because Cargo combines feature requests.

Minimal builds keep input precedence, validation, source records, middleware,
typed state, async execution, injected capabilities, and output capture.
