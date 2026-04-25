# spio-core-process audit shard - 2026-04-22

**Purpose:** Record resolver and process-lifecycle findings from the parallel external audit pass.

**Last updated:** 2026-04-22

Scope: `resolver`, `manifest`, `toolchain`, `process lifecycle`, `state machine`, `timeout`, tests, and gates in `styio-spio`.

## Findings

1. `src/SpioResolve/Resolver.cpp:317-339` used `RunProcess` to capture `git archive` output in memory and then wrote that buffer to disk. `RunProcess` caps stdout at 1 MiB by default, so larger git snapshots could be truncated and fail during extraction.
   Fix applied: switched archive creation to `git archive --output <temp>.tar` so resolver snapshotting no longer depends on captured stdout size. Added `ResolverTests.ResolvesLargeGitSnapshotsWithoutArchiveTruncation`.

2. `src/SpioCore/Process.cpp:185-205` wrote `stdin_text` synchronously before draining stdout/stderr. That leaves a deadlock window for commands that both consume stdin and emit enough output to fill pipe buffers.
   Fix applied: moved stdin writes into the poll loop with non-blocking stdin and added `ProcessTests.StreamsLargeStdinWhileDrainingStdout`.

## Verification

- `cmake --build /home/unka/styio-spio/build-codex --target spio_native_tests -j2`
- `/home/unka/styio-spio/build-codex/bin/spio_native_tests --gtest_filter='ProcessTests.StreamsLargeStdinWhileDrainingStdout:ResolverTests.ResolvesLargeGitSnapshotsWithoutArchiveTruncation'`
- `/home/unka/styio-spio/build-codex/bin/spio_native_tests --gtest_filter='ProcessTests.*:ResolverTests.*'`

## Files changed

- `src/SpioCore/Process.cpp`
- `src/SpioResolve/Resolver.cpp`
- `tests/native/ProcessTests.cpp`
- `tests/native/LockTests.cpp`

## Residual risk

- Resolver/source-build/registry subprocess call sites still do not consistently set explicit timeouts. Long-running or hung external tools can still stall those commands until an outer command-level timeout or manual kill.
