---
name: sagemath
description: Use before writing, editing, or debugging a SageMath (.sage) script, or before running one with `sage`. Covers Sage-specific footguns -- stdout block-buffering when redirected to a file or pipe (a killed job can silently lose everything printed so far), the fix via PYTHONUNBUFFERED or explicit flush, and the `sage -python`/`--python` trap that silently drops the Sage library and preparser.
---

# SageMath scripting: recurring pitfalls

These are traps that have cost real debugging time. Read this before
writing, editing, or launching a `.sage` script, and re-check it when a
long-running job looks stuck with no output.

## Monitoring long-running jobs

- **Sage's stdout fully block-buffers when redirected to a file or pipe.**
  Unlike a terminal, a redirected/piped stdout is not flushed per line by
  default — output accumulates in memory and is only written out when the
  buffer fills or the process exits normally. Verified: `sage script.sage
  > run.log` on a script that prints once every ~4s produced **zero
  bytes** in `run.log` for the entire run, then dumped everything at once
  at exit. **Consequence: a killed job (e.g. `SIGKILL`, or the process
  hitting a hard timeout) can lose every `print` it ever made, even ones
  from minutes earlier.**
  Fix — either works, verified:
  - Launch with `PYTHONUNBUFFERED=1 sage script.sage`. (Verified:
    iterations of the same test script then appeared in the log roughly
    every 4s, in real time, instead of all at once at exit.)
  - Or call `sys.stdout.flush()` explicitly after each `print` you want
    visible during the run.
- **Never pipe a long-running job directly into `tail`.** `cmd | tail -n
  20` shows nothing until `cmd` closes its stdout — i.e. until it
  finishes — because `tail` must see the end of the stream to know what
  "the last N lines" are. This is on top of Sage's own buffering above,
  and applies regardless of it. Redirect to a file instead
  (`PYTHONUNBUFFERED=1 sage script.sage > run.log 2>&1`) and watch the
  file (`tail -f run.log`, or just re-read it).

## Running Sage code non-interactively

- **`sage -python script.sage` (or `--python`) is not "Sage with
  unbuffered output" — it silently drops Sage entirely.** This flag runs
  the bare Python 3 interpreter bundled with Sage: no `sage.all` import,
  no Sage preparser. Verified: a one-line script calling
  `EllipticCurve(...)` fails with `NameError: name 'EllipticCurve' is not
  defined` under `sage -python -u`, while `sage script.sage` (no flag)
  runs it fine. If you need the full Sage environment, always invoke as
  `sage script.sage` (optionally with `PYTHONUNBUFFERED=1` prefixed, as
  above) — never `sage -python`/`--python` for an actual `.sage` file.
