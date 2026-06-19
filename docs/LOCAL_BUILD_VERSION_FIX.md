# Local Build: Fixing the `mlx>=0.31.2` Dependency Conflict

## Symptom

After building this branch locally and running `pip install -e .`, pip prints:

```
ERROR: pip's dependency resolver does not currently take into account all the
packages that are installed. This behaviour is the source of the following
dependency conflicts.
mlx-lm 0.31.3 requires mlx>=0.31.2; platform_system == "Darwin", but you have
mlx 0.31.2.dev20260619+68cf2fdd which is incompatible.
```

## Root cause

A plain `pip install -e .` builds mlx with a **development** version string such as
`0.31.2.dev20260619+68cf2fdd`. Under [PEP 440](https://peps.python.org/pep-0440/)
version ordering, a `.dev` release sorts **before** the corresponding final release:

```
0.31.2.dev20260619  <  0.31.2
```

So although the number looks like `0.31.2`, pip treats it as a *pre-release of*
`0.31.2`, which does **not** satisfy mlx-lm's requirement `mlx>=0.31.2`.

The version string is produced by `get_version()` in `setup.py`: when the
`PYPI_RELEASE` environment variable is unset, it appends `.dev<date>+<githash>`.

```python
# setup.py
pypi_release = int(os.environ.get("PYPI_RELEASE", 0))
dev_release  = int(os.environ.get("DEV_RELEASE", 0))
if not pypi_release or dev_release:
    today = datetime.date.today()
    version = f"{version}.dev{today.year}{today.month:02d}{today.day:02d}"
if not pypi_release and not dev_release:
    # ... append "+<git short hash>"
```

## Fix

Build/reinstall with `PYPI_RELEASE=1`, which produces a clean `0.31.2` version
(no `.dev` suffix, no git hash):

```bash
PYPI_RELEASE=1 pip install -e .
```

## Verification

```bash
$ python -c "import mlx.core as mx; print(mx.__version__)"
0.31.2

$ pip check
No broken requirements found.
```

## Reading the version: `mlx.__version__` is intentionally absent

`mlx.__version__` does **not** exist — and this is correct, *not* a side effect of
the rebuild. The top-level `mlx` is a **namespace package** (there is no
`mlx/__init__.py` in the repo); the real modules are `mlx.core`, `mlx.nn`, and
`mlx.optimizers`. The version lives on `mlx.core`, not on the top-level package.

| Access                                   | Result                          |
|------------------------------------------|---------------------------------|
| `mlx.__version__`                        | ❌ `AttributeError` (by design) |
| `mlx.core.__version__`                   | ✅ `0.31.2`                     |
| `importlib.metadata.version("mlx")`      | ✅ `0.31.2`                     |

Use one of the supported forms:

```python
import mlx.core as mx
print(mx.__version__)              # 0.31.2

from importlib.metadata import version
print(version("mlx"))              # 0.31.2  (same value mlx-lm's >= check uses)
```

Both report `0.31.2`, consistent with the distribution metadata that mlx-lm's
`mlx>=0.31.2` requirement is checked against.

### Gotcha: `import mlx` alone does not give you `mlx.core`

Because `mlx` is a namespace package, a bare `import mlx` binds only the
namespace — it does **not** import the `core` submodule. Accessing `mlx.core`
afterward raises `AttributeError`:

```python
import mlx
mlx.core.__version__
# AttributeError: module 'mlx' has no attribute 'core'
```

You must import the submodule **explicitly** before using it:

```python
import mlx.core as mx
mx.__version__              # '0.31.2'

# or
import mlx.core
mlx.core.__version__       # '0.31.2'
```

The same rule applies to every submodule — `import mlx.nn`,
`import mlx.optimizers`, etc. This is standard MLX usage and is unrelated to the
local build; `import mlx.core as mx` is the form used throughout MLX's own docs
and examples.

## Notes

- The original message is a **warning**, not a hard failure — the mlx install
  itself succeeds and runtime works. The `PYPI_RELEASE=1` build is the clean fix.
- Any future plain `pip install -e .` (without `PYPI_RELEASE=1`) reintroduces the
  `.dev` suffix and the warning. Always use `PYPI_RELEASE=1 pip install -e .` when
  reinstalling this branch.
- If mlx-lm later raises its floor (e.g. `mlx>=0.31.3`), bump
  `MLX_VERSION_PATCH` in `mlx/version.h` accordingly before rebuilding.
