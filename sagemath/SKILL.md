---
name: sagemath
description: Use before writing, editing, or debugging a SageMath (.sage) script, or before running one with `sage`. Covers Sage-specific footguns -- stdout block-buffering when redirected to a file or pipe (a killed job can silently lose everything printed so far), the fix via PYTHONUNBUFFERED or explicit flush, the `sage -python`/`--python` trap that silently drops the Sage library and preparser, `load()` resolving relative paths against the working directory instead of the calling script, running doctests via `sage -python -m sage.doctest` when `sage -t` is missing, variable-name collisions inside Sage's own code (a curve built over a ring whose generator is not named `x` can break Sage's internals with a TypeError that looks like the caller's bug), `Subsets` returning elements from an earlier call in a different parent (e.g. `t` over GF(4) instead of GF(2)), and function-field traps (a reducible defining polynomial silently giving genus -1, `places_infinite()` defaulting to degree 1, `genus()` unavailable on towers, `DrinfeldModule.is_isomorphic` failing over F_q(T)).
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
- **`sage -t` can be missing from an install; run the doctester as
  `sage -python -m sage.doctest` instead.** Verified on a Sage 10.8
  install (`/var/tmp/sage-10.8-current`): `sage -t file.sage` fails with
  `exec: sage-runtests: not found`, while
  `sage -python -m sage.doctest file.sage` runs the standard doctester on
  the same file (34 tests, "All tests passed!"). This is the one
  legitimate use of `sage -python`: the doctest framework imports Sage and
  preparses the `sage:` examples itself, so the trap above does not apply.
  Don't hand-roll a doctest runner before trying this.

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

## Cached constructors can hand back elements from an earlier call

- **`Subsets(L)` can return elements built in a *previous* call, living in
  a different parent, when the old and new elements compare equal.**
  Verified on Sage 10.8:

  ```python
  t4 = GF(4, "a")["t"].gen()
  t2 = GF(2)["t"].gen()        # t4 == t2 and hash(t4) == hash(t2)
  _ = Subsets([t4])
  list(list(Subsets([t2]))[-1])[0].parent()
  # Univariate Polynomial Ring in t over Finite Field in a of size 2^2
  ```

  The two `Subsets` objects are not identical, and `Set([t2])` on its own
  is unaffected, so the caching layer is somewhere inside `Subsets`; what
  matters is the observed behaviour. In a loop over `q = 4, 2` this
  surfaced far from the call, as
  `TypeError: unsupported operand parent(s) for *: 'Finite Field in a of
  size 2^2' and '... over Finite Field of size 2'` inside unrelated
  arithmetic — once again reading as a bug in the caller.
- **Fix: iterate subsets with `itertools.combinations`** (or any plain
  Python construction) when the elements may come from different parents
  across calls; verified to keep the parent of `t2`.
- The general shape: when an element's parent is inexplicably "one from
  before", suspect a cached constructor keyed on equal-comparing
  elements (`t` over `GF(2)` and over `GF(4)` compare equal) before
  suspecting your own code.

## Function fields: silent wrong answers and missing methods

- **`K.extension(f)` accepts a reducible `f` without complaint, and
  `genus()` then returns `-1`.** Verified on Sage 10.8 over
  `K = FunctionField(GF(3))`: `y^2 - x^2`, `(y - x)*(y - x - 1)` and
  `y^2 - 1` all give an "extension" whose `genus()` is `-1`, with no
  error or warning. A negative genus is the only symptom. **Always check
  `f.is_irreducible()` (over `K`) before trusting any invariant of
  `K.extension(f)`.**
- **`places_infinite()` returns only the places of degree 1 by default.**
  Verified on Sage 10.8: for `F = K.extension(y^2 - (2*x^4 + x + 1))`
  over `GF(3)` (the infinite place is inert, so the place above it has
  degree 2), `F.places_infinite()` is `[]`, while `F.places_infinite(2)`
  returns the place. Code that indexes `places_infinite()[0]` then fails
  with `IndexError`, or, worse, a count of `len(places_infinite())` is
  silently wrong. Pass the degree explicitly (`places_infinite(d)` for
  each `d` that can occur) when the infinite place may be inert.
- **`genus()` is not implemented on a tower of function fields; use
  `simple_model()`.** Verified on Sage 10.8: for `F = K.extension(...)`
  over `K = FunctionField(GF(2))` and `L = F.extension(...)`,
  `L.genus()` raises `NotImplementedError: computation of genus over
  non-prime constant fields not implemented yet` -- misleadingly, since
  the constant field is `GF(2)`. `M, _, _ = L.simple_model()` gives the
  same field as a simple extension of `K`, and `M.genus()`,
  `M.places_infinite(d)` work.
