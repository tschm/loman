# CLAUDE.md

Guidance for working in this repository.

## What this is

`loman` — a computation graph that tracks the state of each node and the
dependencies between them, so a change can trigger a partial rather than a full
recalculation. Originally by Ed Parcell (see `AUTHORS`).

Under `src/loman/`:

- `computeengine.py` — the `Computation` object: the public API and the graph
  itself. This is the entry point.
- `nodekey.py`, `consts.py` — node identity and the state enum values.
- `planning.py` — works out what needs recomputing after a change.
- `graph_utils.py`, `util.py` — graph traversal and general helpers.
- `serialization/` — saving and loading a computation. `computation.py` is the
  top level; `values.py` and `transformer.py` handle per-value encoding,
  `blobs.py` large payloads, `compression.py` the zstd layer, `profile.py` and
  `default.py` the configuration of which transformer applies.
- `ui/` — the anywidget-based viewer: `builder.py`, `viewmodel.py`, `value.py`,
  `widget.py`, plus `static/widget.js` and `static/widget.css`.
- `compat.py`, `_extras.py`, `exception.py` — version compatibility, optional
  dependency resolution, and the exception types.

Two dependency facts are load-bearing and documented in `pyproject.toml`:

- `pandas >= 2.0` is a real floor, not caution. Serialization records each
  datetime value's resolution via `Timestamp.unit`/`DatetimeIndex.unit` and
  restores it with `as_unit()`, all of which arrived with non-nanosecond support
  in pandas 2.0. On an older pandas a microsecond index is silently reread as
  nanoseconds — wrong timestamps, not an import error.
- `zstandard >= 0.22` is required, not optional. Saving compresses by default,
  which is only a sensible default because zstd rejects incompressible data fast.
- `pyarrow` is genuinely optional and resolved by name through
  `loman._extras.require`, so there is no import statement for deptry to find.
  That is why `[tool.deptry.per_rule_ignores]` carries `DEP002 = ["pyarrow"]` —
  do not "fix" it by adding a top-level import.

## Ownership: locally owned vs Rhiza-managed

This repo syncs its dev infrastructure from the
[`jebel-quant/rhiza`](https://github.com/jebel-quant/rhiza) template. Note the
**older pointer schema**: `.rhiza/template.yml` uses `template-repository:` and
`template-branch:` (currently `v0.10.3`), not the `repository:`/`ref:` pair newer
repos use. `/rhiza:update` handles either key, but do not "modernise" the file by
hand — the pin and the schema move together.

**The authoritative, machine-generated list of synced files is the `files:` block
of `.rhiza/template.lock`** — when in doubt, consult it.

### Locally owned — edit these freely

- `src/` — the library source, including the shipped `ui/static/` assets
- `tests/` — the test suite and its fixtures
- `pyproject.toml` — project metadata, dependencies, and tool config
- `README.md`, `AUTHORS`, `CHANGELOG.md`, `mkdocs.yml`, `CLAUDE.md`
- `Makefile` — **repo-owned here**, unlike newer rhiza repos. It sets
  `MKDOCS_EXTRA_PACKAGES`, `DEFAULT_AI_MODEL` and `GH_AW_ENGINE`, then
  `include`s `.rhiza/rhiza.mk`. Keep it small; that is what makes it safe to own.
- `.rhiza/make.d/00-additional-deps.mk` and `20-compat-tests.mk` — this repo's
  own make fragments (see below)
- `.readthedocs.yaml`, `docs/`, `scripts/`
- `.rhiza/template.yml`, `.rhiza/.env` — the template pin, and the
  `SOURCE_FOLDER`/`MARIMO_FOLDER` values (`docs/notebooks`, not rhiza's default)

### Rhiza-managed — do NOT edit in place; fix upstream

These are overwritten by the next sync. To change one, open a PR against
`jebel-quant/rhiza` (or exclude the path in `.rhiza/template.yml`), then re-sync:

- `.github/workflows/rhiza_*.yml` — all CI/CD workflows
- `.github/` scaffolding — issue/PR/discussion templates, `dependabot.yml`,
  `release.yml`, rulesets, `secret_scanning.yml`
- `.rhiza/rhiza.mk` and the template's `.rhiza/make.d/*.mk` fragments
- `.pre-commit-config.yaml`, `ruff.toml`, `pytest.ini`, `.bandit`,
  `.editorconfig`, `.python-version`, `cliff.toml`
- `.devcontainer/`, `.claude/commands/rhiza_*.md`
- `LICENSE`, `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, and the
  synced `docs/` pages

## Quality gates

This repo predates rhiza v1.4, so the gates are still the **synced make layer**:
the repo-owned `Makefile` `include`s `.rhiza/rhiza.mk`, which pulls in
`.rhiza/make.d/*.mk`. Run them as bare `make <target>` — never call
`.venv/bin/...` directly. `make help` lists everything, grouped by fragment.

- `make install` — venv and dependencies, via `pre-install` → `install-graphviz`
- `make fmt` — the pre-commit hooks
- `make typecheck` — `ty`
- `make test` — the suite with the coverage gate
- `make coverage-badge` — regenerate `_tests/coverage-badge.svg`
- `make docs-coverage` — interrogate docstring coverage
- `make deptry` — unused/missing dependency analysis
- `make security` — pip-audit and bandit
- `make license` — fail on GPL/LGPL/AGPL
- `make semgrep`, `make suppression-audit`, `make todos` — the quality extras
- `make rhiza-test` — the template's own bundled checks under `.rhiza/tests/`
- `make benchmark`, `make hypothesis-test`, `make stress` — the test extras
- `make book` / `make serve` — build the companion book, and serve it on :8000
- `make marimo`, `make marimo-validate` — the notebooks under `docs/notebooks`
- `make all` — `fmt deptry test docs-coverage security license typecheck rhiza-test`

Two targets are this repo's own, not the template's:

- **`make install-graphviz`** (`.rhiza/make.d/00-additional-deps.mk`) — installs
  graphviz, retrying transient failures, and hangs off `pre-install::` so
  `make install` gets it. `pydotplus` needs the system binary.
- **`make test-compat`** (`.rhiza/make.d/20-compat-tests.mk`) — runs the
  version-sensitive tests against the **oldest supported pandas**. This is the
  gate behind the `pandas >= 2.0` floor above; `make test` alone does not cover it.

Do not reach for `make mutation`. rhiza v1.5.0 stopped offering mutation testing
(Jebel-Quant/rhiza#1492) and the recipe drives a mutmut 2.x CLI that mutmut 3
removed.

## Conventions

- The coverage gate is `COVERAGE_FAIL_UNDER`, default **90** in
  `.rhiza/make.d/test.mk`. This repo does not raise it, but the suite is expected
  to cover every module in `src/loman/` — see the test-layout note below, which
  depends on that.
- `[tool.interrogate]` sets `ignore-overloaded-functions = true`: `@overload`
  stubs delegate their docs to the concrete implementation, so they do not count
  against docstring coverage.
- `[tool.bandit]` skips `B301`, `B403` and `B404` — pickle and subprocess use is
  inherent to serialization here. Do not widen that list casually.
- `MKDOCS_EXTRA_PACKAGES` in the `Makefile` pins `mkdocs_graphviz<2` on purpose:
  in 2.0 it stopped being a Markdown extension and became an MkDocs plugin, and
  the zensical build only shims a fixed set of plugins, so every `dot` code block
  would silently render as a plain fence. Unpin only once zensical can run
  it.
- Two markers are declared: `stress` and `property`.
- Optional dependencies are resolved at run time through `_extras.require`, not
  by top-level import. Follow that pattern for anything new and optional.

## Test layout

Tests live **flat under `tests/`, grouped by behaviour**, not mirroring
`src/loman/`. `[tool.check_test_layout]` records why: `test_serialization.py`
covers the whole `serialization/` package, and `test_typing.py` exercises the
annotated public surface and so has no single source counterpart. The parity that
filename mirroring proxies for is enforced directly by the coverage gate instead
— so **do not reshuffle the suite to mirror the package**.

Groups worth knowing:

- `test_computeengine.py`, `test_planning.py`, `test_containers.py`,
  `test_nodekeys.py`, `test_consts.py` — the graph core.
- `test_serialization.py`, `test_serialization_behaviour.py`,
  `test_serialization_properties.py`, `test_compression.py`, `test_byos.py` —
  the save/load path, including hypothesis properties.
  `tests/fixtures/pandas2.loman` is a stored artefact: it pins that a file
  written by an older version still loads.
- `test_compat.py`, `test_pandas_compat.py`, `test_api_compat.py` — the
  compatibility surface. These are what `make test-compat` exercises against an
  older pandas.
- `test_ui_*.py`, `test_visualization.py` — the widget and rendering path.
- `tests/benchmarks/`, `tests/stress/` — subscription cost/scaling and UI stress,
  behind the `stress` marker and exempt from layout parity by default.
- `tests/fixtures/pipeline.py`, `factory_pipeline.py` — shared computation
  fixtures; `tests/perf.py` is a manual profiling helper, not a test module.
