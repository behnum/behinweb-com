---
title: "Python Just Got Fast(er) — And I'm Still Catching Up"
keywords:
  - Python 3.14
  - Free-Threaded Python
  - No-GIL
  - CPython JIT
  - uv
  - Ruff
  - Astral
  - Rust Tooling
featured: true
tags:
  - Python
  - Performance
  - Developer Tools
description: "A practical catch-up on Python's quiet revolution: the optional no-GIL build (PEP 703/779), CPython's experimental copy-and-patch JIT (PEP 744), and the Rust-powered uv + ruff toolchain that finally makes Python tooling fast."
pubDatetime: 2026-05-24T07:11:28+02:00
modDatetime: 2026-05-24T07:11:28+02:00
slug: python-just-got-fast-nogil-jit-rust-toolchain
---

These days I'm spending most (if not all) of my time working in Javascript, but there's a memory I can't shake. Late 2019, MFT final crunch. I had a Python script chewing through some embarrassingly parallel numerical work on a borrowed 12-core machine, and `htop` showed exactly one core lit up like a Christmas tree while the other eleven sat there filin' their nails. I knew, vaguely, that the GIL was to blame. I knew, vaguely, that PyPy existed. I knew, very vaguely, that there were people on the internet trying to rip the GIL out, and that "Rust-based Python tooling" was a phrase being said in tones I associated with crypto Goolakh bros. I shrugged, rewrote the hot loop with `multiprocessing`, and moved on.

Some six+ yrs later, almost every one of those vague things has _shipped_. And I genuinely think the average Python developer hasn't noticed yet, because the changes arrived in a series of quiet thuds rather than one big bang. So here's the catch-up, written by someone who took the long way around.

## TABLE OF CONTENTS

## The GIL is (optionally) dead

PEP 703 (Sam Gross's proposal to make the Global Interpreter Lock optional) was accepted in 2023. Python 3.13 (October 2024) shipped the first experimental free-threaded build. And then in June 2025, the Steering Council accepted [PEP 779](https://peps.python.org/pep-0779/), and the free-threaded build officially lost its "experimental" tag with Python 3.14. As the [3.14 docs](https://docs.python.org/3/whatsnew/3.14.html) put it: _"The free-threaded build of Python is now supported and no longer experimental. This is the start of phase II where free-threaded Python is officially supported but still optional."_

Two things to internalize:

- It's a **separate build**. The default `python3.14` still has the GIL. The free-threaded one is `python3.14t`. You opt in.
- Single-threaded code in the no-GIL build is **5–10% slower** than the standard build, depending on your workload. That's the price of safe refcounting w/o a global lock — way better than Larry Hastings's Gilectomy days (which paid ~2× slowdown) but still a price. [See here](https://peps.python.org/pep-0703/)

The trick that made it tractable is **biased reference counting**: each object has a "home" thread that can bump refcounts with cheap non-atomic ops; only cross-thread access pays the atomic cos. Combine that with deferred refcounting for immortal-ish objects (modules, types) and a mimalloc-based allocator, and the overhead drops to something the community could live with..

If I'd had this in 2019, I would have saved myself a weekend of `multiprocessing` debugging — and probably a chunk of my MFT test deadline. Try it now:

```bash
uv python install 3.14t
uv venv --python 3.14t
source .venv/bin/activate
python -c "import sys; print(sys._is_gil_enabled())"
# False
```

The catch: your C extensions need to opt in too. NumPy (♡), SciPy, PyTorch, Pillow, Cython .. all done. The long tail of `pip install some-random-package-from-2580` is still… a tail. The community tracks readiness at [py-free-threading.github.io](https://py-free-threading.github.io).

## The JIT exists. It's just not the JIT you were dreaming of.

This is where I keep having to recalibrate my expectations. CPython 3.13 shipped an experimental JIT via [PEP 744](https://peps.python.org/pep-0744/), authored by Brandt Bucher at Microsoft. It uses **copy-and-patch**: at build time, LLVM compiles a library of tiny micro-op "stencils" with holes; at runtime, the JIT stitches stencils together for a hot trace and patches the holes. No LLVM at runtime, no register allocator, no IR pipeline. It's a template assembler, basically · and that's the _point_.

What I keep wanting it to be: PyPy in the box. What it actually is, as of 3.14:

> _"The CPython JIT still has a relatively small effect on this benchmark. Even for the best case... the speedup related to the JIT is only of x1.2, compared to x25 with PyPy!"_ [^1]

The picture isn't uniformly rosy either. Ken Jin, one of the core devs working on it, wrote a candid reflection: _"In the richards benchmark, we see a ~20% speedup, but on the nbody benchmark, we see a ~10% slowdown on my system."_ [^2] Geometric mean across the full benchmark suite is closer to **a few percent**, sometimes barely above noise.

That sounds like a bust. It isn't — but only if you squint at the right thing. PEP 744 explicitly says the JIT will only graduate from experimental when it provides _"a meaningful performance improvement for at least one popular platform (realistically, on the order of 5%)"_. The current goal is **infrastructure**, not speed. Every Tier-2 micro-op the interpreter learns about becomes a stencil the JIT can chain. The compounding starts later.

MFT-test me would have read that and gone "so it's useless?" Engineer me reads it and thinks: this is the same arc as V8 circa 2008 or LuaJIT circa 2005.. a substrate that pays off over multiple releases. I'm leaving `PYTHON_JIT=1` in my shell and getting on with my life.

## The Rust-powered toolchain is the part that _actually_ changed my day-to-day

Here is the part that's not theoretical. Here is the part where I owe Charlie Marsh a beer.

The whole reason I stopped touching Python side projects between roughly 2020 and 2023 was the tooling. To stand up a new project I needed `pyenv` (Python versions), `poetry` or `pip-tools` (deps), `pipx` (tool isolation), `virtualenv`, `black` (format), `isort` (imports), `flake8` (lint), and `pylint` (more lint, slower). They each had opinions. They each had a config file. They each took a noticeable second to start. CI was a wall of yellow warnings about deprecated wheels.

Astral, a tiny company founded by Charlie Marsh in 2022, replaced almost all of it with two binaries written in Rust: **uv** and **ruff**.

### uv

`uv` is "an extremely fast Python package and project manager, written in Rust", a single static binary that replaces pip, pip-tools, pipx, poetry, pyenv, twine, and virtualenv. The benchmark numbers are silly: _"In benchmark tests, uv demonstrates 8–10x faster performance than pip and pip-tools without caching, and an astounding 80–115x faster when running with a warm cache."_ Real Python tested it in the wild and got a more modest but still dramatic result: _"pip installed JupyterLab in 21.409 seconds, while uv did it in 2.618 seconds—about eight times faster."_ [^3]

The first time I ran `uv sync` on a real project I genuinely thought it had silently failed. I scrolled back through the terminal looking for the error. There was no error. It had just... 💨 finished.

The mental model I use:

| Old                               | New                                  |
| --------------------------------- | ------------------------------------ |
| `pyenv install 3.14`              | `uv python install 3.14`             |
| `python -m venv .venv`            | `uv venv`                            |
| `pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| `pipx install ruff`               | `uv tool install ruff`               |
| `poetry add fastapi`              | `uv add fastapi`                     |
| `poetry run pytest`               | `uv run pytest`                      |

`uv.lock` is a universal, cross-platform lockfile resolved with PubGrub — which means its error messages on conflicts are actually readable, unlike pip's "could not find a version that satisfies the requirement" wall of text.

### ruff

`ruff` is the linter and formatter. Same company, same Rust-is-the-secret-weapon story. Its docs claim _"10–100x faster than existing linters (like Flake8) and formatters (like Black)"_ and _"Drop-in parity with Flake8, isort, and Black."_ [Source](https://docs.astral.sh/ruff/) The drop-in part is what makes adoption trivial — FastAPI moved from Black to Ruff in late 2023 ([PR #10517](https://github.com/fastapi/fastapi/pull/10517)), Pydantic followed shortly after.

Config lives in `pyproject.toml` and replaces five separate config files:

```toml
[tool.ruff]
line-length = 100
target-version = "py314"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "RUF"]
# E/F = pycodestyle + pyflakes, I = isort, B = bugbear,
# UP = pyupgrade, SIM = simplify, RUF = ruff-specific
ignore = ["E501"]  # line length handled by formatter
```

Two commands: `ruff check --fix .` and `ruff format .`. That's it. ت That's the whole linting story.

## The bigger picture (and where to be honest)

A pattern is visible if you tilt your head: **Python's hot paths are increasingly written in Rust**. Pydantic v2 has a Rust core. Polars eats Pandas's lunch on most operations. tiktoken, the OpenAI tokenizer (or more accurately, core tokenization engine), is Rust. uv and ruff are Rust. Even Astral's in-progress type checker, `ty`, is Rust.

This isn't a defeat for Python any more than NumPy was a defeat in the 2000s. "Python" has always meant "Python orchestrating fast code in another language." The other language used to be C and Fortran. Now it's Rust .. safer to write, easier to distribute as a single binary, and small enough that a two-person team can ship something that displaces a decade of accumulated toolin'.

Things I'm still skeptical about, for the record:

- **The JIT.** Real wins are 18–24 months out, minimum. Don't refactor for it.
- **Free-threaded dependencies.** Validate your whole tree before flipping the switch in production. The long tail is real.
- **Cold startup.** CPython still takes ~30 ms before your code runs. None of this fixes that.
- **Type checking.** mypy and pyright are still the standard. `ty` is interesting but perhaps not ready.

## What I tell people now

If you've been away from Python for a while like I half was, here is the smallest meaningful catch-up:

1. `curl -LsSf https://astral.sh/uv/install.sh | sh` — install uv. Use it on one project this week.
2. `uv tool install ruff`. delete `black`, `flake8`, `isort`, and `pyupgrade` from your pre-commit config. You will not miss them.
3. `uv python install 3.14t` — run something CPU-bound against it. Watch every core light up. If you're old enough to remember the htop-with-one-core feeling, this part is genuinely emotional. ت

The Python I learned initially was a beautiful language wrapped in mediocre tooling, with a famous performance ceiling. The Python of 2026 has a tooling story that's better than most languages I touch, a path to true parallelism, and the early scaffolding of a real JIT. Maybe I'll spend more time on all this someday.

![FTC](@assets/images/ccores.jpg)

---

## Down the Rabbit Hole

- [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)
- [PEP 779 – Criteria for supported status for free-threaded Python](https://peps.python.org/pep-0779/)
- [PEP 744 – JIT Compilation](https://peps.python.org/pep-0744/)
- [What's New in Python 3.14](https://docs.python.org/3/whatsnew/3.14.html)
- [Ken Jin — Reflections on 2 years of CPython's JIT Compiler](https://fidget-spinner.github.io/posts/jit-reflections.html)
- [News: faster CPython, JIT and 3.14 — discuss.python.org](https://discuss.python.org/t/news-faster-cpython-jit-and-3-14/85326)
- [uv Benchmarks — Astral Docs](https://docs.astral.sh/uv/reference/benchmarks/)
- [uv vs pip — Real Python](https://realpython.com/uv-vs-pip/)
- [Ruff documentation](https://docs.astral.sh/ruff/)
- [FastAPI: Adopt Ruff format (PR #10517)](https://github.com/fastapi/fastapi/pull/10517)
- [Python Free-Threading Compatibility Tracker](https://py-free-threading.github.io)

[^1]: https://discuss.python.org/t/news-faster-cpython-jit-and-3-14/85326

[^2]: https://fidget-spinner.github.io/posts/jit-reflections.html

[^3]: https://realpython.com/uv-vs-pip/

[^4]: https://medium.com/@joedanields/uv-the-next-generation-python-package-manager-thats-revolutionizing-development-workflows-6db37446a465

[^5]: https://github.com/faster-cpython/benchmarking-public

[^6]: https://github.com/faster-cpython/benchmarking-public/blob/main/results/bm-20241214-3.14.0a2+-0ac40ac-JIT/bm-20241214-pythonperf1_win32-x86-python-0ac40acec045c4ce780c-3.14.0a2+-0ac40ac-vs-3.12.0.md

[^7]: https://www.devclass.com/development/2025/07/09/despite-30-months-work-core-developer-says-pythons-jit-compiler-is-often-slower-than-the-interpreter/1629293

[^8]: https://discuss.python.org/t/news-faster-cpython-jit-and-3-14/85326/7
