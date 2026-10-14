# ISIMIP review checklist

Conventions inferred from ISI-MIP/isimip-utils, isimip-qc, isimip-publisher, isimip-ea
and isimip-data (state of October 2026). There is no written style guide behind them,
so use them as defaults: the code in the touched file wins where it differs, and
points marked *(varies)* are handled differently across repos — only flag them when
the diff is inconsistent with its own repo.

## General Python (all repos)

### Language level and formatting

- `requires-python = ">=3.11"`; ruff `target-version` py311/py312. Use modern syntax:
  `X | None`, builtin generics (`list[str]`, `dict[str, Any]`), `tomllib`,
  `collections.abc` for `Iterable`/`Sequence`, `datetime.now(UTC)` instead of the
  deprecated `datetime.utcnow()` (publisher migrated explicitly).
- Line length 120. Single quotes for strings; double quotes only to avoid escaping.
  Inside messages, names are quoted with double quotes in a single-quoted string:
  `'Global attribute "isimip_id" not yet set.'`.
- ruff rule set `B, C4, E, F, I, PGH, RUF, UP, W, YTT` (with `RUF012` ignored).
  `# noqa` must name the code (`# noqa: E711`), as `PGH` enforces.
- Multi-line calls/literals: magic trailing commas and one argument per line in
  repos that run `ruff format` (isimip-utils, isimip-ea, isimip-data). isimip-qc and
  isimip-publisher don't run the formatter and use aligned continuation lines —
  match the file *(varies)*.
- Constant iterables: tuples in isimip-utils (`PROTOCOL_LOCATIONS`), lists elsewhere
  *(varies)* — don't flag either way. Module-level constants are `UPPER_CASE`.

### Imports

- isort sections: future, standard library, [`pytest`], [`django`, `rest_framework`
  in isimip-data], third party, **`isimip_utils` as its own block** (in the tools),
  first party, local folder.
- Inside a package, use relative imports (`from .config import settings`,
  `from ..fixes import fix_set_global_attr`). isimip-data uses absolute
  `isimip_data.<app>` imports across apps and relative imports within an app.
- Import modules from isimip_utils by their submodule
  (`from isimip_utils.cli import ArgumentParser, parse_path`), not via star imports.

### Logging and output

- One module-level `logger = logging.getLogger(__name__)`.
- Lazy %-formatting in log calls: `logger.debug('settings = %s', settings)` — flag
  f-strings or `.format()` inside `logger.*()` calls.
- `print()` is reserved for the actual output of CLI commands (e.g. the publisher's
  `list_*`/`count_*`), not for diagnostics. No leftover `breakpoint()`/`pdb`.
- Logging is configured once, in `main()`, via `isimip_utils.cli.setup_logs`;
  library code (isimip-utils) never configures handlers.

### Errors

- Reuse the exceptions in `isimip_utils.exceptions` (`ConfigError`, `NotFound`,
  `DidNotMatch`, `ValidationError`, `ExtractionError`, `FetchError`), all
  `RuntimeError` subclasses, before adding new ones. Re-raise with `from e`.
- Missing required configuration raises `ConfigError` with a message naming the
  setting (`'ARCHIVE_DIR is not set'`). Some older code raises `RuntimeError` —
  new code should prefer `ConfigError`.
- In CLI entry points, user errors end in `parser.error(...)` with a clear message
  rather than a traceback (see isimip-qc `main.py` for `NotFound`/`FetchError`).
- Don't swallow errors silently. A bare `except:` / `except Exception: pass` or
  returning an empty result on failure needs a reason in a comment (e.g. the
  publisher's `list_files` treats a failing `find` as "no files").

### Naming and structure

- Functions are plain module-level functions; classes are used for models
  (`Dataset`, `File`, `Extraction`, …) and settings. Verb prefixes follow intent:
  `fetch_*` (network, possibly cached), `load_*` (local file), `find_*`, `match_*`,
  `parse_*` (argparse types), `validate_*` (raise `ValidationError`), `compute_*`,
  `init_*`/`create_*`, `write_*`/`update_*`, `check_*`/`fix_*` (isimip-qc).
- Derived, expensive or lazily evaluated attributes use `functools.cached_property`
  (the repos migrated away from Django's and custom variants). Watch for cached
  properties that depend on state that changes later.
- Use `pathlib.Path` for paths (`.expanduser()` for user input, `.as_posix()` when a
  string is needed); avoid string concatenation of paths.
- External tools (`find`, `ssh`, `rsync`, `cdo`, `nccopy`) are called with
  `subprocess` and an **argument list**, never `shell=True` or an interpolated string.
- Type hints and Google-style docstrings (`Args:` with types and defaults,
  `Returns:`, `Raises:`) are **required for public functions in isimip-utils**
  (rendered by mkdocstrings at utils.isimip.org). In the tools they are optional —
  don't ask for them, but new docstrings should use the same style.
- Comments are short, lower-case sentences explaining *why* (`# continue only if at
  least one time step is selected`). A non-obvious workaround needs one.

## ISIMIP domain

- **Never hardcode what the protocol defines** — specifiers, variables, units,
  grids, resolutions, calendars, file name patterns, schemas. Fetch them with
  `isimip_utils.protocol` (`fetch_definitions`, `fetch_pattern`, `fetch_schema`,
  `fetch_tree`) through the configured `PROTOCOL_LOCATIONS`. isimip-qc moved grids
  and dtypes from hardcoded values to the protocol for this reason.
- Protocol JSON is versioned and incomplete for some paths: access optional keys
  with `.get()` and degrade gracefully (warn/skip) instead of crashing.
- Path/file matching goes through `isimip_utils.patterns` (`match_dataset_path`,
  `match_file_path`, specifiers maps); don't re-implement regexes for ISIMIP file
  names. Dirname and filename specifiers must agree.
- File names end in `_<start_year>_<end_year>.nc` (years optional for some data);
  year ranges are **inclusive**. Check off-by-one in year/period handling and time
  slicing.
- NetCDF conventions (see `isimip_utils.netcdf.init_dataset`): dimensions
  `time`/`lat`/`lon`, `NETCDF4_CLASSIC`, fill value and `missing_value` `1e20`,
  time units `days since 1601-1-1 00:00:00`, `proleptic_gregorian` calendar,
  lat in [-90, 90], lon in [-180, 180], lat usually descending. Changes that write
  files must keep these.
- Files are large (GBs, many thousands per dataset): open datasets in a `with`
  block or close them explicitly; avoid loading full variables when a slice or
  streaming works; prefer iterators/generators over building huge lists; long loops
  use `tqdm` progress bars (publisher) or log progress.
- Specifier, identifier and attribute names are lower case snake case as in the
  protocol (`climate_forcing`, `time_step`, `isimip_id`).

## CLI tools (isimip-qc, isimip-publisher, isimip-ea, scripts on isimip-utils)

- Use `isimip_utils.cli.ArgumentParser(prog='isimip-<tool>')`. Defaults can come
  from `isimip.toml`, `~/.isimip.toml`, `/etc/isimip.toml` (section `[isimip-<tool>]`)
  and environment variables `ISIMIP_<DEST_UPPER>` (also from `.env` via `setup_env`).
- New options: kebab-case flag, explicit snake_case `dest`, `help` text (with
  `[default: …]` where useful), sensible `default`; booleans are
  `action='store_true', default=False`.
- **Option types and config files:** `build_default_args` applies the `type` to
  config-file values only for `parse_filelist`, `parse_locations`, `parse_path`,
  `parse_version`; env values get any `type`. A new option with a lambda or custom
  type will therefore receive *untyped* values from `isimip.toml` — flag it, and
  prefer the existing `parse_*` helpers (`parse_list`, `parse_dict`, `parse_path`, …).
- Renaming/removing an option or `dest` silently breaks users' `isimip.toml` files
  and `ISIMIP_*` env vars — needs a README/CHANGELOG note.
- Settings: a module-level singleton `settings = Settings()` subclassing
  `isimip_utils.config.Settings`, filled with `settings.from_dict(vars(args))`;
  keys are UPPER_CASE (`settings.LOCAL_DIR`). Derived settings are UPPER_CASE
  `@cached_property` methods on the subclass (`LOCAL_PATH`, `PATTERN`,
  `DEFINITIONS`). Code reads settings via the singleton, not via passed-around
  `args`. Mutating settings outside `main()`/config is a smell.
- Entry point `main()` in `main.py`, registered in `[project.scripts]`; version via
  `-V/--version` from the package `VERSION`.

### isimip-qc specifics

- Checks are **auto-discovered**: any function named `check_*` in a module under
  `isimip_qc/checks/` (recursively) runs for every file, in sorted module order.
  So a helper accidentally named `check_*` becomes a check, and a new check must not
  rely on another check having run first (`--check <name>` runs it standalone).
- Signature `check_<name>(file)`; report through `file.info/warning/error/critical`
  with %-style args — not via `logger` or `raise`. Use `FileCritical`/`FileError`
  exceptions only to abort processing of a file.
- Severity matters: `critical` blocks copy/move, `error`/`warning` are fixable or
  advisory, `info` for passes and exemptions. A wrong level changes which files get
  published.
- Fixable issues pass `fix={'func': fix_<name>, 'args': (file, ...)}` (or
  `fix_datamodel=…`); fix functions live in `fixes.py`, log what they do via
  `file.info`, and guard for the object's existence with `file.error('Cannot apply
  fix: …')`.
- Checks must survive degenerate input: empty or 1-element axes, missing variables
  or attributes, missing protocol entries.
- User-visible changes get an entry in `CHANGELOG.md` (grouped as in the existing
  release sections).

### isimip-publisher specifics

- Each subcommand is a plain function in `commands.py`, imported in `main.py` and
  added to the matching subparser loop (by positional arguments). A command missing
  from both places is unreachable. Composite commands (`run`, `link`, `init`) chain
  others and share results via the `store` singleton — check that a command still
  works standalone *and* in the chain.
- `--mock` must not copy/modify real data; `--datacite-test-mode` must be honoured
  by anything talking to DataCite.
- Operations touching `PUBLIC_DIR`, `ARCHIVE_DIR` or the database are destructive
  for the live repository: look for missing validation before deletes/moves, and
  for partial state when a loop fails halfway (commit/rollback granularity).
- The SQLAlchemy models in `utils/database.py` define the **metadata database
  schema** read by isimip-data (see `ecosystem.md`). New indexes follow the naming
  `<table>_<column>[_public|_root]_idx`.
- Use SQLAlchemy 2 style query helpers already in use (`with_entities` rather than
  `values`); comparisons to `None`/`True` in column expressions carry
  `# noqa: E711/E712`.

### isimip-ea specifics

- Processing is driven by `Dataset.all()`, `Extraction.gather(...)` and plot classes
  over placeholder permutations (`{model}` etc. via `isimip_utils.parameters`).
  Check that new placeholders are covered in all `*_PARAMETERS`/`*_PERMUTATIONS`.
- Heavy lifting belongs in `isimip_utils.extractions`/`xarray`/`plot`; isimip-ea
  wires it together. Generic new functionality is better added to isimip-utils.
- Respect `--force`, `--fetch-extractions`, `--skip-extractions`, `--skip-plots`
  semantics: existing outputs are reused unless forced.

## isimip-data (Django/DRF backend)

- Layout: `config/settings/base.py` (+ `environments/*`, `local.py`), apps under
  `isimip_data/<app>/` with conventional modules: `admin.py`, `constants.py`,
  `filters.py`, `forms.py`, `managers.py`, `models.py`, `serializers.py`,
  `sitemaps.py`, `templatetags/`, `utils.py`, `views.py`, `viewsets.py`, `tests/`,
  `templates/<app>/` (with `cards/`, `snippets/`, `email/` subfolders),
  `assets/js/{api,components,hooks}`, `assets/scss`. New code should land in the
  matching module.
- Settings: every setting has its default in `config/settings/base.py`
  (app-prefixed, e.g. `METADATA_PAGE_SIZE`); code reads `settings.X` directly,
  without `getattr`/`hasattr` guards.
- **Two databases**: `default` (Django-managed: caveats, annotations, access, …) and
  `metadata` (written by isimip-publisher, models with `managed = False` and
  explicit `db_table`, read-only for Django). Metadata queries use
  `.using('metadata')`; no migrations for unmanaged models; no cross-database joins
  (relations to datasets are stored as UUID arrays, queried with `__overlap`).
- Queries: `select_related`/`prefetch_related` for anything rendered per row,
  `Prefetch(..., queryset=...)` must not accidentally filter related rows, add
  `.distinct()` when filtering across M2M/array joins, avoid unbounded counts
  (`METADATA_MAX_COUNT` paginator). Check templates for per-row queries too.
- Visibility: public/restricted/archived datasets and non-public caveats must stay
  filtered for anonymous and normal users (`.public(request.user)`, `public=True`);
  check new endpoints and querysets for this, and that admin-only actions check
  staff/permissions.
- REST API (`/api/v1/...`, DRF `ReadOnlyModelViewSet` + filter backends +
  `Pagination`) is public and used by isimip-client and external users: changed
  field names/types or URL changes are breaking — call them out.
- Django templates swallow missing attributes: after a model refactor, grep the
  templates for the old names; a broken lookup renders as empty, not as an error.
- Translatable strings use `gettext`/`{% translate %}` where the surrounding code
  does.

## isimip-data frontend (React/JS)

- React 18 function components, one component per `.jsx` file named in PascalCase;
  hooks in `hooks/` (`use*`), data fetching through `@tanstack/react-query` hooks
  in `hooks/queries.js` calling static `fetch*` methods of an `*Api` class in `api/`
  (raising `Error(response.statusText)` when `!response.ok`). Bundled with Vite,
  styled with Bootstrap 5 and per-app SCSS.
- ESLint (enforced): single quotes, double quotes for JSX attributes, no
  semicolons, 2-space indent with `SwitchCase: 1`, max 120 chars, `multiline-ternary:
  never`, `simple-import-sort` groups: react/external → `isimip_data/...` → parent
  modules → parent components → sibling modules → sibling components.
- Reuse shared code from `isimip_data/core/assets/js` (`utils/api` `encodeParams`,
  `hooks/ls`, `hooks/search`, generic components) instead of duplicating it.
- Unused variables must be prefixed with `_`.
- Don't nag about missing frontend tests — the project only has backend tests.

## Testing

- pytest everywhere; tests live in `<package>/tests/test_<module>.py` (isimip-data:
  per app) and test data/fixtures under `testing/`.
- isimip-utils enforces `--cov-fail-under=90` in CI: new public functions/branches
  without tests will fail CI. Network access to the protocol is mocked
  (`tests/helper.py` `mock_json`/`mock_content`).
- isimip-data tests use pytest-django (`db`, `client`), fixtures loaded from
  `testing/fixtures` into both databases, and log in as `user`/`admin` to test
  permission-dependent views — new endpoints should test anonymous, user and admin.
- isimip-publisher tests run CLI commands end-to-end against `testing/` directories
  and a test database (`.env.pytest`); schema changes require updated SQL dumps in
  `testing/sql`.
- Flag missing tests for new backend/library behaviour; don't write or run them.

## Packaging, CI, docs

- `pyproject.toml` with hatchling + hatch-vcs (version from git tags), MIT license,
  classifiers for Python 3.11–3.14, console script in `[project.scripts]`.
- Dependencies: tools pin isimip-utils with extras (`isimip-utils[netcdf]~=2.1.0`).
  **If the diff uses a new isimip-utils function/argument, check the pin allows the
  isimip-utils release that provides it** (and that it's released at all).
- Pre-commit: `check-ast/yaml/json`, `end-of-file-fixer`, `trailing-whitespace`,
  `debug-statements`, ruff (+ `ruff-format` where configured), `typos`, and eslint
  in isimip-data. Spelling exceptions go in `[tool.typos]`/`.typos.toml`.
  Migrations are excluded from pre-commit in isimip-data.
- GitHub Actions `ci.yaml`: tests on PRs to `main`, build, and PyPI trusted
  publishing on tags. Keep `permissions` minimal and `persist-credentials: false`.
- README documents CLI options and setup; isimip-utils has mkdocs reference pages
  under `docs/reference/` — a new public module needs a page there.
