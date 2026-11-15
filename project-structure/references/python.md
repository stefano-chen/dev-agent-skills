# Python

## Baseline layout

A library, and anything else that is built into a distribution and installed, uses the `src/` layout:

```
my-project/
├── pyproject.toml          # metadata, dependencies and tool configuration in one place
├── README.md
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── __main__.py     # optional: enables `python -m my_project`
│       └── ...
└── tests/
    ├── conftest.py
    └── ...
```

- The `src/` layout means tests run against the installed package, not against whatever happens to be in the working directory. That catches packaging mistakes (missing files, wrong imports) that a flat layout hides.
- A flat layout (package directory at the root) is fine for a small application or script collection that is never built into a distribution. A framework with its own layout keeps it (Django's `manage.py` beside the project package). In an existing project, keep whichever layout it has.
- `pyproject.toml` is the single place for metadata and tool settings (ruff, pytest, mypy). Avoid adding `setup.py`, `setup.cfg` or per-tool config files to a project that doesn't already have them.
- Follow the project's existing package manager (uv, Poetry, pip-tools, plain pip). For a new project, `uv init --package` (or `--lib`) produces the layout above.
- Distribution names use hyphens (`my-project`); import names use underscores (`my_project`). Modules and packages are `lowercase_with_underscores`.

## Inside the package

Start with modules, not sub-packages. A handful of well-named `.py` files is easier to navigate than a tree of directories each holding one file. Promote a module to a package when it has grown past a few hundred lines and has clear internal parts.

For an application with distinct features, group by feature:

```
src/shop/
├── __init__.py
├── main.py             # creates the app, includes routers; no logic
├── config.py           # settings loaded from the environment
├── db.py               # engine and session setup
├── orders/
│   ├── __init__.py
│   ├── router.py
│   ├── service.py
│   ├── models.py
│   └── schemas.py
└── catalog/
    └── ...
```

A small service is fine with one `routers.py`, `models.py` and `services.py` each. Move to feature packages when those files start holding several unrelated things.

## Files and naming

- A module is a cohesive group of related classes and functions, not one class. One class per file is a habit from other languages; in Python it produces many tiny modules and imports like `from shop.orders.order import Order`.
- Split a module when it has separable parts that callers import independently, or when its role files (`models.py`, `service.py`) have come to hold several unrelated things. When a module becomes a package, what happens to the old import paths depends on who imports them. In a published library, re-export the public names from the new `__init__.py` so that users' imports keep working. Inside an application, where every importer is in the repository, update the importers and re-export only what is meant to be the package's public surface.
- Name a module for what it holds, as a short lowercase noun (`pricing.py`, `invoices.py`). Follow the project on singular versus plural and on role names (`router.py` or `routes.py`, `schemas.py` or `dto.py`), and use the same role name in every feature.
- Test files are `test_<module>.py`, so pytest discovers them and the test for a module can be found by name.
- Generated code (protobuf and gRPC stubs, API clients) goes in its own clearly named package such as `_generated/`, is excluded from linting and formatting, and is never edited by hand. Alembic migrations stay in the directory Alembic manages.

**Public surface.** A package's `__init__.py` re-exports what other packages are meant to use, and `__all__` can make it explicit. In a library, prefix internal modules and names with an underscore. In an application the feature package's `__init__.py` marks the surface and module names stay plain (`service.py`, not `_service.py`). Keep `__init__.py` free of logic, and keep its re-exports few and cheap to import, because importing any submodule runs it.

**Circular imports** are the usual symptom of dependencies pointing both ways. Fix the direction (move the shared piece down, or have the higher-level module pass what is needed) in preference to hiding the cycle behind function-level imports.

**Module names to avoid:** at the top level, or in a directory that scripts are run from, anything that shadows the standard library or a dependency (`logging.py`, `email.py`, `json.py`, `random.py`), since which module gets imported then depends on the path. Inside a package the name is qualified (`shop.orders.types`), so it does not shadow anything; it is allowed, though a more specific name reads better where one exists. Also `utils.py` and `helpers.py` as catch-alls; name the module for its content.

## Entry points

- Command-line tools are declared under `[project.scripts]` in `pyproject.toml` and point at a function, typically `my_project.cli:main`.
- `__main__.py` and `cli.py` parse arguments and call into the package. Logic stays in importable modules so it can be tested without a subprocess.
- Guard script-style modules with `if __name__ == "__main__":`.

## Tests

- A top-level `tests/` directory, outside the package, is the pytest-recommended arrangement with the `src/` layout. Mirror the package structure (`tests/orders/test_service.py` for `src/shop/orders/service.py`) so the test for any module is easy to find.
- Mirroring gives test files in different directories the same basename (`tests/orders/test_service.py`, `tests/catalog/test_service.py`), which pytest's default import mode rejects with an "import file mismatch" error. In a new project set `addopts = "--import-mode=importlib"` under `[tool.pytest.ini_options]`. In an existing project keep its solution, which is often `__init__.py` files in the test directories.
- Shared fixtures go in `conftest.py` at the narrowest directory level that covers their users.
- When unit and integration tests need to run separately, either split them into `tests/unit/` and `tests/integration/` or use pytest markers. Follow what the project already does.
- Test data lives under `tests/` (for example `tests/data/`), not in the package.

## Configuration and data

- Settings are loaded from the environment in one module (pydantic-settings or similar) and passed inward. Scattered `os.environ` lookups make it impossible to see what the application needs.
- `.env` files are for local development and are not committed; commit a `.env.example`.
- Non-code files the package needs at runtime live inside the package and are read with `importlib.resources`, so they still work when installed.
- `scripts/` holds operational one-offs that are not part of the package. `notebooks/` and `data/` stay out of `src/`.

## Framework notes

- **Django**: follow its conventions (`manage.py`, a project package with settings, one app per cohesive domain created with `startapp`). Apps are Django's feature packages; a `core` or `common` app that everything depends on is the junk drawer to watch for. Split `settings` into a package only when environments really differ.
- **FastAPI / Flask**: the feature layout above; one `APIRouter` or blueprint per feature, registered in `main.py` or an app factory.
- **Libraries**: keep the public API small and explicit in the top-level `__init__.py`, ship a `py.typed` marker if the package is typed, and keep internal modules underscore-prefixed.
- **Data and ML projects**: keep reusable code in the package under `src/`, and treat notebooks as consumers of it. Pipelines, configs and experiment outputs get their own top-level directories, with outputs ignored by version control.

## Workspaces

Use one package until there is a concrete need for more: separately published libraries, or several deployables sharing code. uv workspaces (or the equivalent in the project's tool) keep multiple packages in one repository, each with its own `pyproject.toml`, typically under `packages/` and `apps/`.

## Common problems to look for in an audit

- Imports that only work from a particular working directory, or `sys.path` manipulation.
- Top-level modules shadowing the standard library or a dependency.
- `utils.py` or `common/` holding unrelated functions that most of the codebase imports.
- Circular imports worked around with function-level imports.
- Logic in `__init__.py`, `__main__.py` or at module import time.
- One class per file throughout, or a single `models.py` or `services.py` of several thousand lines shared by unrelated features.
- Tests that depend on being run from the repository root, or test helpers inside the shipped package.
- Configuration read from the environment in many modules.
- Several competing config files (`setup.py`, `setup.cfg`, `requirements*.txt`, `pyproject.toml`) describing the same project differently.
