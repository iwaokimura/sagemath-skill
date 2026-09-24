---
name: sagemath
description: Use before writing, editing, or debugging a SageMath (.sage) script, or before running one with `sage`. Covers Sage-specific footguns -- stdout block-buffering when redirected to a file or pipe (a killed job can silently lose everything printed so far), the fix via PYTHONUNBUFFERED or explicit flush, the `sage -python`/`--python` trap that silently drops the Sage library and preparser, `load()` resolving relative paths against the working directory instead of the calling script, and variable-name collisions inside Sage's own code (a curve built over a ring whose generator is not named `x` can break Sage's internals with a TypeError that looks like the caller's bug).
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
- **`load("helper.sage")` resolves against the current working
  directory, not against the script that calls it.** Verified on Sage
  10.8: a script containing `load("lib.sage")` works when run from its own
  directory and fails with `OSError: did not find file 'lib.sage' to load
  or attach` when run as `sage /path/to/main.sage` from elsewhere. Fix —
  verified: anchor the path at the script itself,

  ```python
  import os
  load(os.path.join(os.path.dirname(os.path.abspath(__file__)), "lib.sage"))
  ```

  `__file__` is defined when running `sage main.sage`; it points to the
  preparsed `main.sage.py`, which Sage writes *next to the script* (so add
  `*.sage.py` to `.gitignore` in a repository of `.sage` files).

## Variable names are not local: Sage's internals assume `x`

- **A curve built over a polynomial ring whose generator is not named
  `x` can break Sage's own code, with an error that names neither the
  variable nor the curve.** Verified on Sage 10.7, the two runs
  differing *only* in the generator's name:

  ```python
  K = Qp(11, 6)
  R = PolynomialRing(QQ, 'X'); v = R.gen()          # or 'x'
  H = HyperellipticCurve(v**5 - 1).change_ring(K)
  H.coleman_integrals_on_basis(H.lift_x(K(2)), H.lift_x(K(6)))
  ```

  With `'X'` this raises `TypeError: unsupported operand parent(s) for
  -: 'Univariate Polynomial Ring in x over 11-adic Field ...' and
  'Univariate Polynomial Ring in X over 11-adic Field ...'`; with `'x'`
  it returns the integrals. The cause is inside Sage
  (`hyperelliptic_padic_field.frobenius` builds `x^p` in a ring of its
  own naming and subtracts it from the curve's polynomial), so the
  traceback points at Sage's source and reads as a bug in your code —
  which is what makes it expensive.
- **Fix: name the generator `x`** whenever an object will be handed
  back to Sage's own algorithms (`PolynomialRing(QQ, 'x')`). If `x` is
  already taken in your script, bind the generator to another *Python*
  name — `Rx = PolynomialRing(QQ, 'x'); X = Rx.gen()` — since what
  matters is the ring's variable name, not your local one.
- The general shape: when a Sage routine fails on parents that "should"
  match, compare the *variable names* printed in the two parents before
  looking anywhere else.
