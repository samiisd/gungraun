# Standalone binary benchmark design

## Existing execution path

Rust benchmark declarations produce `api::BinaryBenchmarkGroups`, which the
benchmark executable sends to `gungraun-runner` using its version-coupled binary
protocol. The runner combines configuration, builds commands, runs tools, parses
native profiles, calculates totals and regressions, and emits summaries. The
normal metadata constructor may invoke Cargo to locate workspace paths.

The standalone frontend parses a separate, versioned TOML model and maps it to
the same `BinaryBenchmarkGroups`. A dedicated metadata constructor uses the
manifest directory and output directory directly. There is no benchmark harness
executable, Cargo metadata query, generated Rust, or consumer build. The
existing process, parser, aggregation, absolute-limit, and output code remains
the sole measurement implementation.

## Public boundary

`gungraun-runner run --config PATH` captures all declared cases and identifies
uncapped cases. `check` additionally requires a positive absolute `Ir` cap on
every case before execution and only reports a guard pass after validating the
complete current result set. Both commands reject missing or zero current `Ir`,
unexpected identities, duplicate results, and unsuccessful execution. Equality
with the cap passes through the existing strict-greater-than limit check.

Version 1 supports a command, literal arguments/environment, cwd, expected exit
code, optional named Callgrind entry point, and absolute `Ir` limits. Nested
defaults merge by field; environment overrides merge by key. Unknown fields,
unsupported metrics, duplicate IDs and invalid limits are errors. Stable ASCII
IDs of at most 100 bytes avoid the existing output name sanitization and
200-byte truncation changing their identity. Case-only duplicate IDs are
rejected to avoid collisions on case-insensitive filesystems.

The frontend exposes a small CLI rather than flattening the harness CLI. It does
not accept filters, baseline loads, test mode, alternate tools or
behavior-changing `GUNGRAUN_*` environment options. These could weaken the
completeness guarantee or change the measurement selected by a reviewed
manifest. Logging/color controls remain available. Existing Rust APIs, their
uncapped benchmarks, and the harness protocol are unchanged.

## Results and failure

The cap applies to the existing Callgrind profile total across its current
process/part/thread records. No per-operation scaling or independent result
parser is added. Validation examines typed current metrics and identities, never
console text or a prior summary. Historical profiles remain available for the
runner's ordinary comparisons but cannot satisfy completeness.

The native Callgrind sanitizer rejects repeated process/part/thread headers
before renaming files, so duplicate measurements cannot overwrite each other
before typed validation. Sanitization failures retain the original profiler
files through the ordinary output lifecycle.

Before execution, summaries for the declared cases are removed so an early
failure cannot leave a previous summary masquerading as a current result.
Current profiles/logs and summaries for successfully parsed measurements are
retained on failure. Execution errors retain the runner's fail-fast policy; cap
regressions do not stop independent cases. A failed aggregate never prints a
guard pass. Limits are read-only inputs and are never automatically raised.
Configuration errors occur before summary cleanup and leave existing files as
historical evidence without accepting them as current results.

## Scope and alternatives

The separate user model prevents an unstable internal wire representation from
becoming the public configuration schema. `toml` is a runner-only dependency,
reusing a version already in the workspace lockfile, with parsing, Serde, and
standard-library support only. A second executable would duplicate
installation/distribution plumbing without isolating a different execution
engine; explicit subcommands fit the existing runner's dispatch boundary.

Upstream [issue #563] proposes TOML defaults for existing CLI settings, while
[VISION.md] lists direct ad-hoc binary benchmarking. Neither specifies this
declarative case model or strict gate. The implementation leaves broader CLI
configuration and ad-hoc profiling for separate design work.

Future schema versions can add deliberately specified settings by mapping them
to existing runner types. Setup callbacks, arbitrary scripts, dynamic fixture
generation, build orchestration, additional profilers, and automatic baseline
approval are outside version 1. Runner installation, target compilation, and
consumer harness compilation are separate costs; this change removes the last
requirement, without claiming an unmeasured cold-CI speedup.

[issue #563]: https://github.com/gungraun/gungraun/issues/563
[VISION.md]: https://github.com/gungraun/gungraun/blob/main/VISION.md
