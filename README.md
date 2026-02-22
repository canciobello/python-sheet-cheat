# python-sheet-cheat

A repo to document decisions, conventions, and "known good" workflows for Python tooling.

---

## uv

**uv** is a fast, modern Python toolchain (package manager + project workflow) that can also **download and manage Python interpreters** for you.

Official docs: https://docs.astral.sh/uv/

---

## Configure uv to use only managed Python interpreters

### Why "managed-only" is useful

Using uv-managed interpreters (instead of distro Python like `/usr/bin/python*`, Homebrew Python, etc.) gives you:

- **Reproducibility**: install the same Python minor/patch on any machine or CI.
- **Isolation from OS changes**: system updates won’t silently change the interpreter your project uses.
- **Easy multi-version testing**: keep 3.9 + latest stable side-by-side without fighting the OS package manager.
- **Cleaner projects**: you can pin/share interpreter intent via config and/or `.python-version`.

### User-wide config location

- **Linux / macOS**: `~/.config/uv/uv.toml`  
  (or `$XDG_CONFIG_HOME/uv/uv.toml` if `XDG_CONFIG_HOME` is set)
- **Windows**: `%APPDATA%\uv\uv.toml`

### Create the config file (Linux/macOS)

```bash
mkdir -p ~/.config/uv
cat > ~/.config/uv/uv.toml <<'EOF'
python-preference = "only-managed"
python-downloads  = "manual"     # downloads only on `uv python install ...`
EOF
```

What these settings do:

- `python-preference = "only-managed"`  
  uv will **never select system Python** for commands that need an interpreter.
- `python-downloads = "manual"`  
  uv will **not auto-download** Python when it’s missing. You must install explicitly via `uv python install ...`.

> Practical implication: with these settings, if you haven’t installed a managed Python yet, project commands
> that need an interpreter will fail until you run `uv python install ...`.

---

## Listing Python interpreters (managed vs system)

### Quick reference

| Goal                                                           | Command                                                            |
| -------------------------------------------------------------- | ------------------------------------------------------------------ |
| List interpreters uv can use (installed + download candidates) | `uv python list`                                                   |
| Only installed interpreters                                    | `uv python list --only-installed`                                  |
| Only **system** interpreters (unmanaged)                       | `uv python list --no-managed-python --only-installed`              |
| Only **uv-managed** interpreters                               | `uv python list --managed-python --only-installed`                 |
| Show more versions/platforms                                   | `uv python list --all-versions` / `uv python list --all-platforms` |

Notes:

- `uv python list` shows **installed interpreters** plus **a curated set of available downloads**.
- `--only-installed` is the most reliable way to check what you actually have installed.

---

## Testing that managed-only is working

### Step A — Baseline (before creating `uv.toml`)

```bash
uv python list
```

You should see some combination of:

- system interpreters (paths like `/usr/bin/python3.12`, `C:\...\python.exe`)
- managed download candidates (shown as "download available" or similar)

Output varies by OS, uv version, and what you already have installed.

### Step B — After setting `python-preference = "only-managed"`

After creating `~/.config/uv/uv.toml` with `python-preference = "only-managed"`:

- uv will **ignore system interpreters for selection**
- `uv python list` will still show managed download candidates
- `uv python list --managed-python --only-installed` will show only installed managed Pythons (possibly empty)

If you have **no managed Python installed yet**, this should be empty:

```bash
uv python list --managed-python --only-installed
# (no output)
```

But you should still see available managed downloads with:

```bash
uv python list --managed-python
```

---

## Install managed Pythons and verify

### Install a managed interpreter

If your goal is "install the latest stable CPython that uv supports":

```bash
uv python install
```

If you want a specific version line:

```bash
uv python install 3.14     # installs latest available 3.14.x
uv python install 3.9      # installs latest available 3.9.x
# or an exact patch (when available):
uv python install 3.14.3
```

### Verify

```bash
uv python list --managed-python --only-installed
```

Optional helpers (paths):

```bash
uv python dir
uv python dir --bin
```

---

## Updating uv-managed Python patch versions (CPython)

```bash
# (optional) update uv so it knows about the newest available patch builds
uv self update

# upgrade installed managed CPython versions to the latest supported patch release
uv python upgrade --preview-features python-upgrade

# verify
uv python list --managed-python --only-installed
```

**Note on `--preview-features`:** This does **not** imply installing alpha/beta/RC Python builds. It only enables
uv’s _patch-upgrade mechanism_, which is marked as preview/experimental. You still get stable releases unless you
explicitly request a pre-release version.

**One more nuance:** available Python versions are _frozen per uv release_, so updating uv can be necessary before
it can "see" a newly released patch.

---

## Creating a packaged project (library or packaged app/CLI) with `uv`

This section documents how to create a **packaged** project (buildable/distributable as a Python package),
either as:

- a **library** (imported by other packages), or
- a **packaged app/CLI** (installs a command via `[project.scripts]`)

> `uv init` creates the skeleton. The project venv (`.venv`) and lockfile (`uv.lock`)
> are created lazily during the first `uv sync` / `uv run` / `uv lock`.

---

### 1) Pick the right project flavor

#### A) Library (meant to be imported by other packages)

```bash
uv init --lib trapipy_log
```

Notes:

- `--lib` implies `--package` (libraries are always packaged).
- Uses `src/` layout and adds `py.typed` for type consumers.

#### B) Packaged app/CLI (meant to expose a command)

```bash
uv init --package trapipy-log
# (same intent, explicit form)
uv init --app --package trapipy-log
```

Notes:

- `--app` projects are the default, and by default they are **not** packaged.
- Adding `--package` makes the app **distributable/installable** and uses a `src/` project structure.
- Packaged apps include a `[project.scripts]` entry point scaffold.

#### C) Default app (not packaged)

```bash
uv init trapipy-log
```

Notes:

- Creates `main.py` and `.python-version`, but **does not** set up a build system.
- This is fine for scripts/services you run with `uv run`, but not for "install as a package".

---

### 2) Specify a Python version during project creation (optional, recommended)

```bash
uv init --lib trapipy_log --python 3.14
# or for packaged CLI:
uv init --package trapipy-log --python 3.14
# short:
uv init --lib trapipy_log -p 3.14
```

`--python/-p` sets which interpreter uv uses to determine the project’s **minimum supported Python version**
(i.e., what goes into `requires-python`), and influences the created `.python-version` pin.

---

### 3) What `.python-version` is used for

By default uv creates `.python-version` containing the **minor version** of the discovered interpreter.
Subsequent uv commands will prefer that version.

Disable pin creation:

```bash
uv init --lib trapipy_log --no-pin-python
```

Change the pin later:

```bash
uv python pin 3.14
# or pin an exact patch
uv python pin 3.14.3
```

---

### 4) Review and fill `pyproject.toml` (important)

After `uv init`, open `pyproject.toml` and set real metadata:

- `[project]`: `name`, `version`, `description`, `requires-python`, `authors`, `license`, `readme`, etc.
- Library-specific: ensure `requires-python` reflects what you actually support (not just what you run locally).
- CLI-specific: ensure `[project.scripts]` points to your `module:function`.

---

### 5) Add dependencies (updates lock + environment)

```bash
uv add typer sqlalchemy
```

What happens:

- Dependencies are added to `pyproject.toml`.
- uv updates `uv.lock` and syncs the environment (creating `.venv` if needed).

Explicit sync:

```bash
uv sync
```

---

### 6) Sanity-check

```bash
uv run python -c "import typer, sqlalchemy; print('ok')"
```

For CLI projects (after `[project.scripts]` is correct):

```bash
uv run trapipy-log --help
```

---

### 7) Updating dependencies in a `uv` project

Here we describe **two** common "upgrade" workflows:

1. **Upgrade everything within your existing version constraints** (safe, routine)
2. **Bump `pyproject.toml` constraints to newer versions** (intentional API churn)

#### Workflow A — Upgrade within the existing `pyproject.toml` constraints

Use this when you want the newest versions **allowed by your current specifiers**.

```bash
uv sync --upgrade
```

What it does:

- updates `uv.lock` to newer compatible versions (respecting `pyproject.toml`)
- syncs your virtual environment to match the updated lock

If you also use dependency groups / extras

```bash
uv sync --upgrade --all-groups --all-extras
```

#### Workflow B — Bump `pyproject.toml` constraints to newer versions

`uv` will not automatically rewrite your `pyproject.toml` just because newer packages exist.
If you want your **declared dependency bounds** to move forward, you must update them.

> Note: `pyproject.toml` only lists **direct dependencies**. Transitives remain in `uv.lock`.

##### Step 1 — Choose how `uv add` writes version bounds (recommended)

Set a policy once:

```toml
[tool.uv]
add-bounds = "major"  # common default
```

Typical choices:

- `major`: allow upgrades within the same major series
- `minor`: tighter bounds (same minor series)
- `lower`: only set a lower bound
- `exact`: pin exactly

##### Step 2 — Re-write your direct dependency specifiers

Re-add your existing dependencies by name. `uv add` will:

- update the version specifier in `pyproject.toml`
- refresh `uv.lock` accordingly

Runtime deps:

```bash
uv add <dep1> <dep2> ...
```

Dev deps (example group name: `dev`):

```bash
uv add --group dev <dep1> <dep2> ...
```

##### Step 3 — Sync

```bash
uv sync
```

## How `uv sync` installs projects (package vs. file-path)

### What counts as "a packaged project"

A repository is treated as a **package project** when `pyproject.toml` defines a **PEP 517 build system**, e.g.:

```toml
[build-system]
requires = ["uv_build>=0.10.4,<0.11.0"]
build-backend = "uv_build"
```

That is the signal that the project can be built into a wheel and **installed into the environment**.

### What `uv sync` does when a build system exists

When you run `uv sync` in a project like this:

1. Creates/updates the project virtual environment (typically `./.venv/`).
2. Resolves & installs dependencies (from `uv.lock` if present).
3. Installs **this project itself** into the venv.

That final step is what makes the project **importable from anywhere inside the venv** (not only runnable by
a relative file path).

> Control knob: you can force or disable "install the project" behavior via `tool.uv.package`
> in `pyproject.toml` (useful for edge cases and migration).

### What "installed" means mechanically

After installation you’ll typically see, inside the venv:

- `.../site-packages/<your_module>/` _(or an editable redirect to it)_
- `.../site-packages/<dist-name>-<version>.dist-info/` _(metadata: name, version, dependencies, entry points, etc.)_

### Editable (default) vs non-editable (`--no-editable`)

#### Editable install (default for `uv sync`)

- The project is installed in **editable mode**.
- **Source code stays in your working tree** (usually under `src/<module>/...`).
- The venv contains a small redirect so imports resolve to your source tree:
  - commonly a `*.pth` file in `site-packages/` that adds your repo’s `src/` to `sys.path`,
  - or an equivalent import hook mechanism (backend-dependent).
- Result: editing files under `src/` affects behavior on the next run/import (you typically restart the process).

#### Non-editable install (`uv sync --no-editable`)

- uv builds a **regular wheel** and installs it normally.
- Package files are **copied into `site-packages/`**.
- Result: editing your repo files does nothing until you rebuild/reinstall (run `uv sync` again).

### How the CLI script gets created (`[project.scripts]`)

When `pyproject.toml` contains:

```toml
[project.scripts]
trapipy-log = "trapipy_log.main:app"
```

Installation registers a **console script entry point** and generates a launcher in the venv, typically:

- Linux/macOS: `./.venv/bin/trapipy-log`
- Windows: `./.venv/Scripts/trapipy-log.exe` (and/or `.cmd`)

That launcher loads the entry point from installed metadata and executes it.
For Typer, `app` is usually a `typer.Typer()` instance (Click runs it as a command).

### Quick checks (what’s actually happening)

#### Where is the package imported from?

```bash
python -c "import trapipy_log; print(trapipy_log.__file__)"
```

- Editable: path points into your repo (e.g., `.../src/trapipy_log/__init__.py`)
- Non-editable: path points into the venv (e.g., `.../.venv/.../site-packages/trapipy_log/...`)

#### Where is the CLI installed?

```bash
# Linux/macOS
command -v trapipy-log
trapipy-log --help
```

Windows (PowerShell):

```powershell
Get-Command trapipy-log
trapipy-log --help
```

## libs and packages conventions, layout, and CLI patterns

This section records the conventions used in **trapipy-log** and presents them as a reusable pattern for other libraries in your Python cheat sheet.

The project is a Python package (`src/trapipy_log/`) with:

- a deliberately small public API (`make_logger`)
- an optional CLI (`trapipy-log`) implemented with Typer for database utilities

---

### Naming and entry points

- **Distribution name (PyPI / installer):** `trapipy-log`
- **Import name (Python module):** `trapipy_log`
- **Console script:** `trapipy-log`

The console script is wired in `pyproject.toml`:

```toml
[project.scripts]
trapipy-log = "trapipy_log.main:app"
```

---

### Layout at a glance

```
trapipy-log/
  pyproject.toml
  README.md
  LICENSE
  src/
    trapipy_log/
      __init__.py        # curated public API (re-exports)
      core.py            # composition/wiring for the library API
      handlers.py        # concrete logging handlers (DB/file/etc.)
      filters.py         # LogRecord enrichment (task/caller, etc.)
      formatters.py      # formatting helpers/formatters
      models.py          # SQLAlchemy models (DB schema)
      main.py            # Typer app (CLI entry)
      py.typed           # PEP 561 marker for typed packages
```

---

### Public API strategy: `__init__.py` is the curated "front door"

`src/trapipy_log/__init__.py` exists to provide a stable, discoverable import path for the _supported_ surface area of the package:

```py
from trapipy_log import make_logger
```

A few practical rules make this work well:

- `__init__.py` should be mostly a **re-export hub** (keep it lightweight and side‑effect free).
- Use `__all__` as the "what we support" list. It’s documentation and tooling guidance, not access control.
- Public names are **not underscored**. Leading underscores signal internal/private intent.

In trapipy-log today, the public API is intentionally minimal:

- `make_logger` is re-exported from `core.py`
- `__all__ = ["make_logger"]` documents that decision

#### What "public" means here

Python cannot prevent users from importing internal modules. They can still do:

```py
from trapipy_log.handlers import DBSyncHandler
```

The convention is:

- **Public API** = documented + re-exported + stable
- **Internal modules** = importable, but allowed to change without notice unless you explicitly promote them

---

### Module boundaries: themed modules + a composition layer

This project follows a simple separation:

- **Themed modules** hold the primitives:
  - `handlers.py` → handlers
  - `filters.py` → LogRecord enrichment
  - `formatters.py` → formatting logic
  - `models.py` → persistence schema
- **`core.py`** holds the orchestration:
  - "wire the primitives together into one coherent behavior"
  - implement the library’s main entry points (`make_logger`)

#### Why `core.py` exists (and what it is not)

`core.py` is an **integration layer**, not a junk drawer.

A good `core.py` contains code that:

- composes several pieces across modules
- defines defaults and high-level policy
- represents the "primary way" users interact with the library

In trapipy-log, `make_logger()` is exactly that: it builds a ready-to-use logger by composing handlers, filters, and formatting behavior.

#### Promoting internals to public without churn

If a component becomes part of the supported API later (e.g., you decide `DBSyncHandler` should be public):

1. Keep the implementation where it naturally belongs (`handlers.py`).
2. Re-export it from `__init__.py`.
3. Add it to `__all__`.
4. Document it and add minimal tests that treat it as a contract.

This avoids moving files around and keeps responsibilities clear.

---

### CLI architecture: `main.py` is the CLI boundary

`src/trapipy_log/main.py` is the **CLI entry module**:

- defines the Typer `app`
- registers commands and options
- keeps the boundary between "library" and "CLI" explicit

A healthy dependency direction is:

- CLI imports library code
- library code does _not_ import CLI code

#### Where command implementations should live

`main.py` is a perfectly good place to keep **all** of your CLI app wiring **and** command bodies **as long as** the code is genuinely _CLI-shaped_:

- parsing / validation of CLI inputs
- prompts / confirmations
- formatting output for humans
- exit codes and user-facing error messages
- simple orchestration of underlying library calls

In other words: **`main.py` can host the full CLI implementation** when the logic is tightly coupled to the command-line UX.

##### When code should NOT live in `main.py`

Move code out of `main.py` when it:

- is **reusable** outside the CLI (e.g., could be called from Python code, another interface, or a background job),
- has a **weak relationship** to the CLI (it’s really just part of the library),
- represents **domain logic** (business rules, persistence, transformations, calculations),
- would make sense **even if Typer didn’t exist**.

A useful rule of thumb:

> **If the code still makes sense without Typer, it probably doesn’t belong in `main.py`.**

Put that logic in its natural home (modules that describe the domain), e.g.:

- `db.py`, `storage.py`, `services/*.py`, `formatters.py`, `logging.py`, `models.py`, etc.

Then the CLI becomes a thin adapter layer that turns CLI input into library calls and formats the results.

##### Scaling up: splitting the CLI like any other app

If the CLI grows large, split it for maintainability (just like any other application):

- keep `main.py` as **entry point + glue** (app creation, global options, command registration)
- move groups/commands into modules, e.g.:
  - `commands/db.py`
  - `commands/inspect.py`
  - `commands/admin.py`
  - `cli_utils.py` (CLI-only helpers: rendering tables, shared prompts, consistent error formatting)

Then register commands from `main.py`:

```py
# main.py
from .commands.db import create_sqlite_db

app.command("create-sqlite-db")(create_sqlite_db)
```

This keeps the CLI discoverable from one entry point while keeping reusable logic in the library modules where it naturally belongs.

##### Testing is not a blocker

Keeping commands in `main.py` is **not** a testing problem. Typer provides strong testing support via `typer.testing.CliRunner`.

Typical pattern:

```py
from typer.testing import CliRunner
from mypkg.main import app

runner = CliRunner()

def test_help():
    result = runner.invoke(app, ["--help"])
    assert result.exit_code == 0
    assert "Usage" in result.output
```

Whether commands live in `main.py` or in `commands/*.py`, you can test the CLI surface the same way. The main difference is architectural clarity: reusable logic belongs in library modules; CLI glue belongs with the CLI.

---

#### Shell completion: what gets installed and when it works

Typer provides completion helpers automatically (via Click). You’ll typically use:

```bash
trapipy-log --install-completion
```

#### Where completion is installed

Completion setup is written to your **shell configuration** (e.g., `~/.bashrc`, `~/.zshrc`, etc.). It is not "installed into the venv".

Completion _works_ when the command is on your `PATH` in that shell session:

- If `trapipy-log` is only available inside an activated venv, completion works **only when the venv is active**.
- If `trapipy-log` is installed as a global/always-on tool, completion works in every session.

If your command isn’t always present, consider add a guard around the completion activation in your shell config:

- "only enable completion if `trapipy-log` exists"

This avoids errors in sessions where the venv/tool isn’t available.

---

# Ruff + Pyright in a `uv` project (VS Code workflow)

This walks you through setting up **VS Code**, installing **Ruff** and **Pyright** with **uv**, and wiring everything together so developers get fast linting, formatting, and strict type checking directly in the editor and in CI.

---

## 1) Install VS Code + common Python extensions

### Install VS Code

Download and install Visual Studio Code from:

- https://code.visualstudio.com/Download

### Install the most common extensions for Python development

Open VS Code → **Extensions** (left sidebar) → search and install:

1. **Python** (Microsoft)
   - Enables Python interpreter selection, debugging, test discovery, and general Python IDE features.
2. **Pylance** (Microsoft)
   - Provides fast IntelliSense and type checking (powered by the Pyright type checker engine).
3. **Ruff** (Ruff / Astral)
   - Runs Ruff inside VS Code for linting, formatting, and quick fixes.

> Why these three? Together, they cover the typical "modern Python editor stack": interpreter + language features (Python), type checking (Pylance/Pyright), and lint/format (Ruff).

---

## 2) Install Ruff + Pyright in your `uv` project

From the project root (where `pyproject.toml` lives), run:

```bash
uv add --dev ruff pyright
```

What this does:

- Adds **ruff** and **pyright** as **development dependencies**.
- Updates the lockfile (`uv.lock`).
- Installs them into your project’s environment (`.venv` if that’s how your repo is set up).

### What each tool does

**Ruff** is a very fast Python **linter and formatter**. It flags common bugs, style issues, unused imports/variables, and many other code-quality problems—and it can auto-fix a large subset of them. In practice, it gives developers immediate feedback and consistent formatting with minimal runtime cost.

**Pyright** is a fast Python **static type checker**. It analyzes your code (without running it) to catch type errors early: wrong return types, invalid attribute access, missing `None` checks, incompatible function arguments, and more. In strict mode it helps keep large codebases predictable and reduces runtime surprises.

---

## 3) Ruff configuration (in `pyproject.toml`)

Add (or keep) the following:

```toml
[tool.ruff]
line-length = 100
target-version = "py314"
exclude = [".git", ".venv", "env", "__pycache__", "build", "dist"]

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "C4", "SIM", "RUF", "N", "PTH", "RET", "PT"]
ignore = []
fixable = ["ALL"]
unfixable = ["F401"]

```

---

## Meaning of each setting

### `[tool.ruff]`

- `line-length = 100`  
  Sets the maximum preferred line width (used by formatting and certain lint rules). A slightly wider limit than 88 (Black’s default) can reduce line wrapping while still keeping code readable.

- `target-version = "py314"`  
  Tells Ruff which Python version you’re targeting so it can apply version-aware rules and modernizations correctly (for example, safe syntax upgrades that depend on the Python version).

- `exclude = [".git", ".venv", "env", "__pycache__", "build", "dist"]`  
  Explicitly lists directories for Ruff to ignore. While Ruff has sensible defaults, defining these explicitly ensures the linter won’t waste time scanning or accidentally modifying auto-generated build artifacts, caches, or virtual environments.

### `[tool.ruff.lint]`

- `select = ["E", "W", "F", "I", "UP", "B", "C4", "SIM", "RUF", "N", "PTH", "RET", "PT"]`  
  Chooses which rule "families" Ruff enforces. This is a comprehensive, high-value set:
  - `E` / `W` / `F`: core style, warnings, and correctness rules commonly associated with **pycodestyle** and **Pyflakes** (syntax errors, undefined names, unused variables, and minor stylistic warnings).
  - `I`: import sorting rules (similar to **isort** behavior).
  - `UP`: **pyupgrade**-style suggestions (modernize syntax when it’s safe for your target version).
  - `B`: **flake8-bugbear** rules (common footguns and suspicious patterns).
  - `C4`: comprehension optimizations (more idiomatic/faster comprehensions).
  - `SIM`: simplifications (reduce unnecessary complexity in expressions/conditionals).
  - `RUF`: Ruff-native rules (Ruff-specific "hygiene" checks), especially useful to prevent stale suppressions (e.g., unused `# noqa`).
  - `N`: **pep8-naming** conventions (ensures classes are CamelCase, functions are snake_case, etc.).
  - `PTH`: **flake8-use-pathlib** (flags older `os.path` patterns and suggests modern `pathlib` equivalents).
  - `RET`: **flake8-return** (encourages cleaner return statements, e.g., avoiding unnecessary `else` after `return`).
  - `PT`: **flake8-pytest-style** (enforces consistent, modern pytest testing practices).

- `ignore = []`  
  An empty list ready for exceptions. Keeping it here is a convenient placeholder for when you inevitably need to disable a specific rule globally that conflicts with your codebase.

- `fixable = ["ALL"]`  
  Allows Ruff to auto-fix any issue that Ruff knows how to fix **when you ask it to** (e.g., via `ruff check --fix`). This does not auto-change code by itself; it just enables fixes for all fixable rules.

- `unfixable = ["F401"]`  
  Overrides the `fixable` setting for specific rules. Setting this to `F401` (unused imports) stops Ruff from automatically deleting imports you just typed but haven’t used yet, which prevents frustrating disruptions while actively writing code.

## 4) Pyright configuration (in `pyproject.toml`)

```toml
[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.14"

venvPath = "."
venv = ".venv"

include = ["src", "tests"]
extraPaths = ["src"]

exclude = ["**/node_modules", "**/__pycache__", "**/.*"]

reportMissingImports = "error"
# Let Ruff handle unused variables to avoid redundant warnings
reportUnusedVariable = "none"
```

### Meaning of each setting (detailed)

- `"typeCheckingMode": "strict"`  
  Enables Pyright’s strictest set of checks. This increases signal (more issues found early) at the cost of requiring better annotations and more explicit handling of edge cases (like `None`). This is typically what teams want for long-lived projects.

- `"pythonVersion": "3.14"`  
  Declares the Python language version the type checker should assume. This affects type system features and standard-library typing details (for example, how certain typing constructs behave based on version).

- `"venvPath": "."` and `"venv": ".venv"`  
  Tells Pyright where to find your virtual environment:
  - `venvPath` is the base folder containing the venv.
  - `venv` is the name of the venv directory within that path.

  With these values, Pyright expects the environment at `./.venv` and uses it to resolve installed packages and stubs.

- `include`: typically `["src", "tests"]`. If you don’t want to type-check tests, remove `"tests"`.

- `extraPaths: ["src"]`: required for `src/` layout so import resolution matches runtime.

- `exclude`: we explicitly exclude common non-source directories to prevent analysis on them.
  - Important: when you specify `exclude`, Pyright won’t apply its automatic default excludes, so we include
    `**/node_modules` and `**/.*` (hidden directories) explicitly.
    This keeps the analysis focused on your source code and avoids noise and performance costs.

- `reportMissingImports`: set to `"error"` to fail CI if imports can’t be resolved (often means missing deps or
  misconfigured import paths).

- `reportUnusedVariable`: set to `"none"` if Ruff owns unused-variable hygiene (Ruff `F` rules). If you prefer
  redundancy, change it to `"warning"` or `"error"`.

---

## 5) VS Code configuration (critical step)

Create (or edit) `.vscode/settings.json` in your repo:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.analysis.autoImportCompletions": true,
  "ruff.enable": true,

  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

### What this does

- **Selects the project interpreter**
  - `"python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python"` points VS Code at your `uv` environment so tooling runs against the same dependencies developers use in the terminal.
  - On Windows, the interpreter path is typically:
    - `${workspaceFolder}\.venv\Scripts\python.exe`

- **Improves import completion**
  - `"python.analysis.autoImportCompletions": true` helps VS Code suggest imports as you type.

- **Ensures the Ruff extension is active**
  - `"ruff.enable": true` When you install the official astral-sh.ruff extension in VS Code, it is enabled by default. Because it defaults to true under the hood, explicitly writing "ruff.enable": true in your settings.json isn't strictly necessary for a fresh install. By explicitly hardcoding "ruff.enable": true into your workspace .vscode/settings.json, you create a workspace override. It guarantees that no matter what global settings a developer has on their personal machine, Ruff will always be forced "ON" when they open your specific repository.

- **Formats and fixes on save using Ruff**
  - `"editor.defaultFormatter": "charliermarsh.ruff"` explicitly tells VS Code to use Ruff for formatting, ignoring other installed formatters like Black.
  - `"editor.formatOnSave": true` enables formatting every time a developer saves a file.

  `codeActionsOnSave`: VS Code recently updated the syntax for editor.codeActionsOnSave. While _true_ still technically works for backward compatibility, the modern and correct value is "explicit".
  - `"source.fixAll.ruff": "explicit"` applies Ruff’s safe auto-fixes.
  - `"source.organizeImports.ruff": "explicit"` lets Ruff organize imports (so you don’t need a separate import tool).

---

## 6) How Pylance relates to Pyright, and Ruff relates to the VS Code extensions

### Pylance and Pyright

- **Pyright** is the type checker engine (the thing that performs static type analysis).
- **Pylance** is the VS Code language extension that provides IntelliSense and type diagnostics in the editor, and it uses the **Pyright engine** under the hood.

Practical takeaway:

- When you see type errors in VS Code via Pylance, you’re effectively seeing Pyright’s analysis.
- Installing `pyright` in the project (via `uv add --dev pyright`) also lets you run the same checks in CI and locally from the terminal.

### Ruff and the VS Code Ruff extension

- **Ruff** is the CLI tool that lints and formats your code.
- The **VS Code Ruff extension** integrates Ruff into the editor:
  - showing lint errors as you type
  - providing "quick fix" actions
  - running formatting and import organization on save (when configured)

Practical takeaway:

- Developers get instant feedback in VS Code, and the same rules run in the terminal/CI.
- One toolchain, consistent results everywhere.

---

## 7) Common developer commands (run via `uv`)

Run these from the project root:

```bash
# Lint
uv run ruff check .

# Auto-fix what Ruff can fix
uv run ruff check . --fix

# Format
uv run ruff format .

# Type check (Pyright)
uv run pyright
```

That’s it: Ruff handles lint + formatting, and Pyright handles type checking, all using your `uv` environment and surfaced directly inside VS Code.

## libs and packages conventions, layout, and CLI patterns

This section records the conventions used in **trapipy-log** and presents them as a reusable pattern for other libraries in your Python cheat sheet.

The project is a Python package (`src/trapipy_log/`) with:

- a deliberately small public API (`make_logger`)
- an optional CLI (`trapipy-log`) implemented with Typer for database utilities

---

### Naming and entry points

- **Distribution name (PyPI / installer):** `trapipy-log`
- **Import name (Python module):** `trapipy_log`
- **Console script:** `trapipy-log`

The console script is wired in `pyproject.toml`:

```toml
[project.scripts]
trapipy-log = "trapipy_log.main:app"
```

---

### Layout at a glance

```
trapipy-log/
  pyproject.toml
  README.md
  LICENSE
  src/
    trapipy_log/
      __init__.py        # curated public API (re-exports)
      core.py            # composition/wiring for the library API
      handlers.py        # concrete logging handlers (DB/file/etc.)
      filters.py         # LogRecord enrichment (task/caller, etc.)
      formatters.py      # formatting helpers/formatters
      models.py          # SQLAlchemy models (DB schema)
      main.py            # Typer app (CLI entry)
      py.typed           # PEP 561 marker for typed packages
```

---

### Public API strategy: `__init__.py` is the curated "front door"

`src/trapipy_log/__init__.py` exists to provide a stable, discoverable import path for the _supported_ surface area of the package:

```py
from trapipy_log import make_logger
```

A few practical rules make this work well:

- `__init__.py` should be mostly a **re-export hub** (keep it lightweight and side‑effect free).
- Use `__all__` as the "what we support" list. It’s documentation and tooling guidance, not access control.
- Public names are **not underscored**. Leading underscores signal internal/private intent.

In trapipy-log today, the public API is intentionally minimal:

- `make_logger` is re-exported from `core.py`
- `__all__ = ["make_logger"]` documents that decision

#### What "public" means here

Python cannot prevent users from importing internal modules. They can still do:

```py
from trapipy_log.handlers import DBSyncHandler
```

The convention is:

- **Public API** = documented + re-exported + stable
- **Internal modules** = importable, but allowed to change without notice unless you explicitly promote them

---

### Module boundaries: themed modules + a composition layer

This project follows a simple separation:

- **Themed modules** hold the primitives:
  - `handlers.py` → handlers
  - `filters.py` → LogRecord enrichment
  - `formatters.py` → formatting logic
  - `models.py` → persistence schema
- **`core.py`** holds the orchestration:
  - "wire the primitives together into one coherent behavior"
  - implement the library’s main entry points (`make_logger`)

#### Why `core.py` exists (and what it is not)

`core.py` is an **integration layer**, not a junk drawer.

A good `core.py` contains code that:

- composes several pieces across modules
- defines defaults and high-level policy
- represents the "primary way" users interact with the library

In trapipy-log, `make_logger()` is exactly that: it builds a ready-to-use logger by composing handlers, filters, and formatting behavior.

#### Promoting internals to public without churn

If a component becomes part of the supported API later (e.g., you decide `DBSyncHandler` should be public):

1. Keep the implementation where it naturally belongs (`handlers.py`).
2. Re-export it from `__init__.py`.
3. Add it to `__all__`.
4. Document it and add minimal tests that treat it as a contract.

This avoids moving files around and keeps responsibilities clear.

---

### CLI architecture: `main.py` is the CLI boundary

`src/trapipy_log/main.py` is the **CLI entry module**:

- defines the Typer `app`
- registers commands and options
- keeps the boundary between "library" and "CLI" explicit

A healthy dependency direction is:

- CLI imports library code
- library code does _not_ import CLI code

#### Where command implementations should live

For small CLIs, keeping command bodies in `main.py` is fine.

As the CLI grows, keep `main.py` focused on:

- Typer wiring (commands, options, help text, grouping)
- minimal glue (argument conversion, formatting output)

And move substantial logic to modules that describe the work:

- `db_ops.py`, `inspect_ops.py`, `cli_utils.py`, etc.

Then register those functions from `main.py`, e.g.:

```py
# main.py
from .db_ops import create_sqlite_db

app.command("create-sqlite-db")(create_sqlite_db)
```

This keeps the CLI discoverable in one file while preventing duplication and keeping the "real work" testable without invoking the CLI.

---

#### Shell completion: what gets installed and when it works

Typer provides completion helpers automatically (via Click). You’ll typically use:

```bash
trapipy-log --install-completion
```

#### Where completion is installed

Completion setup is written to your **shell configuration** (e.g., `~/.bashrc`, `~/.zshrc`, etc.). It is not "installed into the venv".

Completion _works_ when the command is on your `PATH` in that shell session:

- If `trapipy-log` is only available inside an activated venv, completion works **only when the venv is active**.
- If `trapipy-log` is installed as a global/always-on tool, completion works in every session.

If your command isn’t always present, consider add a guard around the completion activation in your shell config:

- "only enable completion if `trapipy-log` exists"

This avoids errors in sessions where the venv/tool isn’t available.

---

# SSH agent autostart + key loading (Windows 11, Ubuntu Server 24.04)

Goal: have an SSH agent available after reboot, and have your key loaded so `git`, `uv`, and other **non-interactive** commands don’t get stuck on a passphrase prompt.

---

## Windows 11 (built-in OpenSSH)

### Start ssh-agent on every boot

Run **PowerShell as Administrator**:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

### Add a key

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
ssh-add -l
```

### After restart (refresh)

Check if it’s still loaded:

```powershell
ssh-add -l
```

If empty, re-add:

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

**Passphrase keys:** after a reboot, the key is encrypted on disk and must be **unlocked once** (you type the passphrase) the first time it’s used. If a background job is the _first_ thing to use it, it can’t type the passphrase → use an unattended deploy key (see bottom).

---

## Ubuntu **Server** 24.04 (recommended: `keychain`)

### Why this is needed on Ubuntu Server

- No desktop keyring service is running to auto-start an agent for you.
- Each SSH login / terminal session is isolated; starting an agent in one session doesn’t help the next session.

`keychain` ensures exactly **one** ssh-agent runs, and every login shell reuses it.

### Step 1 — Install keychain

```bash
sudo apt update
sudo apt install keychain
```

### Step 2 — Hook it into your login shells

Edit `~/.bashrc`:

```bash
nano ~/.bashrc
```

Add this at the very bottom (adjust key filename if different):

```bash
eval $(keychain --eval --agents ssh id_ed25519)
```

### Step 3 — Reload + unlock once

Apply to the current terminal:

```bash
source ~/.bashrc
```

On first run, it will prompt:

- `Enter passphrase for /home/<you>/.ssh/id_ed25519:`

Type it once. After that, your key stays loaded in the agent and future sessions reuse it (so `uv`/`git` won’t prompt again).

### Step 4 — Verify end-to-end (GitHub example)

```bash
ssh -T git@github.com
```

Expected:

- `Hi <user>! You've successfully authenticated, but GitHub does not provide shell access.`

Once you see that, `uv` and `git` SSH operations should work cleanly.

---

## Unattended automation (no prompts after reboot)

If you need jobs to work when nobody can type a passphrase, use a dedicated deploy key **with no passphrase** (least privilege):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_deploy -N ""
chmod 600 ~/.ssh/id_ed25519_deploy
```

Optionally load both keys via keychain:

```bash
eval $(keychain --eval --agents ssh id_ed25519 id_ed25519_deploy)
```

# Working with private GitHub repos as uv dependencies

This document covers two practical workflows when your dependencies live in **private GitHub repositories** and you manage projects with **uv**.

> Note on names: the **distribution name** used in dependency declarations can contain dashes
> (e.g., `trapipy-log`), while the **importable module** is often underscore-normalized
> (e.g., `import trapipy_log`). uv will resolve the dependency name from the repo’s metadata.

---

## Workflow 1 — Add a dependency from a private GitHub repo (SSH)

### Windows note: make sure Git uses the right `ssh.exe`

On Windows, `uv add git+ssh://...` delegates to Git for fetching sources. Depending on your setup, Git might use an SSH implementation that **does not pick up the key you expect**, which can cause authentication failures.

A common fix is to force Git to use Windows OpenSSH:

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

This helps ensure Git uses the same SSH key configuration you already have for GitHub.

> Tip: You can also configure host-specific keys in `~/.ssh/config` (recommended when you use multiple GitHub identities).

### Add the dependency pinned to a tag

Example (private repo over SSH):

```bash
uv add "git+ssh://git@github.com/canciobello/trapipy-log.git" --tag v0.1.0
```

What this does:

- **Resolves the package name** from the repository (from the dependency’s `pyproject.toml` metadata).
- Adds that package name to your **runtime dependencies**:
  - `pyproject.toml` → `[project].dependencies`
- Records the Git source in:
  - `pyproject.toml` → `[tool.uv.sources]` (with `git = ...` and `tag = ...`)
- Updates (or creates) the lockfile:
  - `uv.lock` (pins the full resolved set of packages)

So you end up with a **clean dependency declaration** ("I depend on package X") plus a **source override** ("package X comes from this Git repo + tag").

### Prefer commit pinning for maximum reproducibility

Tags can be moved (even if they _shouldn’t_ be). If you want maximum reproducibility, pin to an immutable commit:

```bash
uv add "git+ssh://git@github.com/canciobello/trapipy-log.git" --rev <COMMIT_SHA>
```

This will record `rev = "<COMMIT_SHA>"` under `[tool.uv.sources]`.

---

## Workflow 2 — Develop a dependency locally while testing it from the dependent project

This workflow is for the common case:

- You’re building **a tool/app** (the "consumer" project).
- It depends on **a library** in a separate repo (e.g., `trapipy-log`).
- You want to edit the library and immediately see the effect in the tool **without** constantly rebuilding/publishing/tagging.

### Side-by-side repos + editable local path during dev, then tag/commit for release

#### 0) Clone both repos next to each other

Example:

```text
~/code/
  trapipy-log/
  windows-organizer/     # your tool/app
```

#### 1) In the tool repo, depend on the library (baseline: pinned Git tag)

```bash
cd ~/code/windows-organizer
uv add "git+ssh://git@github.com/<ORG>/trapipy-log.git" --tag v0.1.0
```

This:

- adds the library to `[project].dependencies`
- records the Git source under `[tool.uv.sources]`
- updates `uv.lock`

> If you want maximum reproducibility, prefer a commit pin:
>
> ```bash
> uv add "git+ssh://git@github.com/<ORG>/trapipy-log.git" --rev <COMMIT_SHA>
> ```

#### 2) When you need to hack the library while developing the tool: switch to an editable local path

```bash
uv add --editable ../trapipy-log
```

This switches the source for that dependency to a **local editable install**. You’ll typically see something like:

```toml
[project]
dependencies = ["trapipy-log"]

[tool.uv.sources]
trapipy-log = { path = "../trapipy-log", editable = true }
```

Conceptually, this is the "editable dev" flow (`pip install -e` / Pipenv-style):

- you edit the dependency repo,
- the dependent project imports from your working tree.

#### 3) Iterate normally (no reinstall loop)

Once the dependency is editable, **any Python code change you make inside `../trapipy-log/` is visible immediately** to `windows-organizer` the next time you run it.

You do **not** need to re-run `uv add`, reinstall, or rebuild just to pick up those code edits.

Typical loop:

```bash
# in the tool repo
uv run pytest
# or
uv run <your-cli> ...
```

Notes:

- If you change **Python code** in the editable dependency: it’s picked up immediately.
- If you change the dependency’s **dependencies** (its `pyproject.toml`): run `uv sync` in the tool repo to update the environment/lock as needed.

#### 4) Promote the library change to a real dependency update (tag/commit)

When your library change is ready, make a proper release pointer in the library repo:

```bash
cd ~/code/trapipy-log
git commit -am "Fix XYZ"
git tag v0.1.1
git push --tags
```

Back in the tool repo, switch the dependency source back to a released tag:

```bash
cd ~/code/windows-organizer
uv add "git+ssh://git@github.com/<ORG>/trapipy-log.git" --tag v0.1.1
uv sync
```

Or pin to a commit:

```bash
uv add "git+ssh://git@github.com/<ORG>/trapipy-log.git" --rev <COMMIT_SHA>
uv sync
```

Result:

- your tool is pinned to a **real version** again (good for CI, colleagues, and installs on fresh machines)
- the editable path override is gone

#### 5) CI rule of thumb: use `--locked` to fail fast

In CI, you typically want reproducibility and fast failure when the lockfile is out of date:

```bash
uv sync --locked
# or
uv run --locked pytest
```

`--locked` will error instead of silently updating `uv.lock`.

---

## How `--editable` works under the hood (the `.pth` file)

When you do:

```bash
uv add --editable ../trapipy-log
```

uv installs that dependency in **editable mode**. Editable mode means:

- your environment keeps the dependency "installed" (metadata is present, `importlib.metadata` sees it, etc.)
- but Python imports the code **from your working tree** (your checkout), not from a copied wheel inside `site-packages`

That’s why edits appear "instantly": you edit the source files, then the next `uv run ...` imports those updated files directly.

### The common mechanism: `site-packages/*.pth`

A very common way to implement editable installs is via a **`.pth` file** inside `site-packages`.

Where you’ll typically find it:

- **Windows venv**: `.venv\Lib\site-packages\*.pth`
- **Linux/macOS venv**: `.venv/lib/pythonX.Y/site-packages/*.pth`

What it does:

- A `.pth` file is plain text.
- At interpreter startup, Python’s `site` machinery reads `.pth` files.
- Any line that looks like a filesystem path is appended to `sys.path`.

So the `.pth` file effectively says: "also import from `../trapipy-log` (or its `src/` directory)".

### What changes require `uv sync`

Editable mode makes **code edits** immediate, but it doesn’t automatically change your environment when you change packaging metadata:

- If you change `trapipy-log`’s declared dependencies / extras / entry points in `trapipy-log/pyproject.toml`, run `uv sync` in the consumer project to refresh the environment and lockfile.
- If your dependency includes compiled extensions or generated artifacts, you may need an explicit rebuild step depending on the build backend.

---

## Practical note: "I don’t want to commit local paths"

Today, the editable-path override lives in `pyproject.toml` under `[tool.uv.sources]`.
uv does not currently provide a clean, "local-only sources override" mechanism that keeps path overrides out of the committed `pyproject.toml`.

So the clean habit is:

- use the editable-path source **locally and temporarily**
- before committing changes to the tool/app repo, switch back to a **Git tag or commit** source
