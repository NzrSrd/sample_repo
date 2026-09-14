# sample_repo

A test fixture for UpgradePilot, a dependency upgrade risk and migration planning tool.

This tree is not an application. Nothing here is meant to run, and several files
are wrong on purpose. UpgradePilot reads a repository and reports what a
dependency upgrade will break. This is the repository it reads in its tests.
Every file exercises one branch of the analyzer's confidence grading for a
Pydantic v1 to v2 upgrade.

## Do not edit this copy

The canonical fixture lives in the UpgradePilot repository at
`backend/tests/fixtures/sample_repo`, beside the builder that assembles it,
`backend/tests/fixtures/repo_builder.py`. The test suite reads that copy.

This repository is a snapshot of the builder's output, published so the tree can
be cloned and browsed on its own. No test reads it. An edit here changes no
assertion. To change the fixture, edit the canonical tree and its expectations
in the same commit.

## How this snapshot differs from the canonical tree

`build_sample_repo(tmp_path)` copies the canonical tree into a temp directory
and makes two changes before committing. Both are already applied here:

1. `src/app/broken.py.txt` is renamed to `src/app/broken.py`. The canonical tree
   stores it under `.txt` so ruff and pytest in the UpgradePilot repository
   never try to parse it.
2. `MAX_INVOICES = 100` is appended to `src/app/models.py` and committed
   separately.

The tree is otherwise identical to the canonical fixture, byte for byte.

## Why there are two commits

The history is `initial import`, then `add invoice cap`. The second commit
touches only `src/app/models.py`, which is also the file the analyzer flags as
affected. That overlap is the point. It gives the churn signal something to
find, so a test can assert that the most affected file is also the most recently
changed one.

`MAX_INVOICES` is never read. It exists to give the second commit something to
change. Do not remove it as dead code.

## What each file is for

| File | Grade | What it exercises |
| --- | --- | --- |
| `src/app/models.py` | High | Pydantic v1 declarations: `BaseModel`, `class Config`, `@validator`, and an `Optional` field with no default. |
| `src/app/service.py` | Medium | Model method calls on annotated parameters, with `pydantic` in scope: `.dict()`, `.parse_obj()`, `.schema()`, `.copy()`. |
| `src/app/consumer.py` | Low | Imports `Customer`, so it is a candidate, but `summarise` takes an unannotated argument. The analyzer cannot resolve the receiver of that `.dict()` call. |
| `src/app/util.py` | None | A false positive trap. `Bag.dict()` is a plain method and no model library is in scope. Grading this call medium is a bug. |
| `src/app/broken.py` | Skipped | Deliberately unparseable. The analyzer must record a `SkippedFile` with a reason instead of raising. Its first comment names `pydantic` so the byte prefilter still selects the file. |
| `src/app/__init__.py` | | Empty. Makes `app` a package. |
| `tests/test_models.py` | | Passes `nickname=None` explicitly, so it holds under both v1 and v2. |
| `pyproject.toml` | | Declares `pydantic>=1.10,<2`. |
| `requirements.txt` | | Pins `pydantic==1.10.13`. |

Both manifests exist so the analyzer can be tested on a repository that declares
a range in one file and a pin in another.

## The expectations that bind this tree

`repo_builder.py` exports these constants, and `tests/unit/test_fixture_repo.py`
asserts each one against the built tree:

| Constant | Value |
| --- | --- |
| `EXPECTED_PYTHON_FILES` | `7` |
| `EXPECTED_HIGH_CONFIDENCE_SYMBOLS` | `("BaseModel", "Config", "Optional", "validator")` |
| `EXPECTED_MEDIUM_CONFIDENCE_SYMBOLS` | `("copy", "dict", "parse_obj", "schema")` |
| `EXPECTED_LOW_CONFIDENCE_SITE` | `("src/app/consumer.py", "dict")` |
| `EXPECTED_UNPARSEABLE` | `"src/app/broken.py"` |
| `EXPECTED_DECLARED_SPECIFIER` | `">=1.10,<2"` |
| `EXPECTED_PINNED_VERSION` | `"1.10.13"` |

`tests/analysis/test_analyzer_end_to_end.py` asserts these tuples for equality
against the analyzer's real output, not containment, so dropping a symbol from
one fails there rather than passing quietly as a smaller claim.

Gutting a fixture file is the failure these tests are built to catch. Three of
the constants were once unasserted, and emptying `models.py`, emptying
`service.py`, or widening the specifier to `>=2.0` each left the suite green.
Add a constant only together with the assertion that binds it.

## The Pydantic v1 idioms in this tree

`src/app/models.py`:

| v1 | v2 |
| --- | --- |
| `class Config` | `model_config = ConfigDict(...)` |
| `orm_mode = True` | `from_attributes=True` |
| `allow_mutation = False` | `frozen=True` |
| `@validator("email")` | `@field_validator("email")` |
| `nickname: Optional[str]` with no default | Same annotation, but required. v1 defaults it to `None`. |

`src/app/service.py`:

| v1 | v2 |
| --- | --- |
| `.dict()` | `.model_dump()` |
| `.parse_obj()` | `.model_validate()` |
| `.schema()` | `.model_json_schema()` |
| `.copy()` | `.model_copy()` |

## Linting

The UpgradePilot repository excludes this tree from ruff and mypy in its
`pyproject.toml`. The code here is invalid on purpose, so a clean lint run over
it would mean the fixture had stopped doing its job.
