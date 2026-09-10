---
name: xtesting-pytest-integration
description: Use when integrating your own Pytest test suites into an Xtesting-based container — install pytest in the Dockerfile, get the test files into the image, declare pytest cases in testcases.yaml with the dir arg and options dict, and collect results.html/results.xml/stdout.log output.
---

# Running your own Pytest tests in Xtesting

The default `pytest` driver (already registered in Xtesting's `setup.cfg` as an
`xtesting.testcase` entry point) runs any pytest suite in-process — no custom
Xtesting driver needed. A pytest hook collects per-test results and the driver
scores the campaign.

## 1. Make the pytest files available in the image

Because tests run in-process, the `dir` you point at must exist inside the
container and be importable if your tests import local modules:

- **pip package (recommended):** ship tests inside your Python package, e.g.
  `mypkg/tests/test_foo.py` → referenced via
  `/usr/lib/python3.12/site-packages/mypkg/tests/` (**match the image's Python
  version** — e.g. `/usr/lib/python3.14/...` on Alpine 3.24).
- **COPY layer:** `COPY tests/ /tests/` then use `/tests` as `dir`.

> pbr note: `pip3 install /src` only ships non-`.py` files when the source
> tree is a git repo **with committed files** (`pbr` reads `git ls-files`).
> Build from a committed checkout or a released wheel, or your `.py` tests
> themselves are still shipped (they are package files) but any data
> fixtures aren't.

## 2. Install the dependency in the Dockerfile

`pytest` (and `pytest-html`) are already declared in xtesting's
`requirements.txt` — present on the official `opnfv/xtesting` image and on any
`pip3 install /src` build. The explicit install is only needed when building
from scratch:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git && \
    pip3 install --break-system-packages --no-cache-dir pytest && pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

## 3. Declare the test case in testcases.yaml

Driver name: `pytest`. Mandatory arg: `dir` (a test file or directory).
Optional: `options` — a **dict** flattened into pytest CLI `--key value`
flags (single-letter keys become `-k`). It always appends `--html`, `--junitxml`,
`-p no:cacheprovider` and (unless set) `--tb no`.

```yaml
tiers:
  - name: pytest
    testcases:
      - case_name: my_pytest_case
        project_name: myproject
        criteria: 100
        blocking: true
        run:
          name: pytest
          args:
            dir: /usr/lib/python3.14/site-packages/mypkg/tests/test_foo.py
            options:
              maxfail: 2
              verbose: ''
```

Packaged sample (case `nineth`, as shipped for the official Python 3.12
image — adjust the path to your container's Python):

```yaml
run:
  name: pytest
  args:
    dir: /usr/lib/python3.12/site-packages/xtesting/samples/fourth.py
```

`options` are rendered with `pytest.main(args=[dir, '-p', xtesting.hook, ...])`
— keys map to flags (`maxfail: 2` → `--maxfail 2`), so anything accepted on
the pytest CLI works.

## 4. Scoring and results

The driver runs `pytest.main()` and:

- Scores `result = 100 * passed / (passed + failed)`; skipped tests are not
  counted.
- Collects per-test details into `details['tests']` (name, status, failure
  traceback for failures).
- Writes `res_dir/results.html`, `res_dir/results.xml` (junit) and
  `res_dir/stdout.log`.
- Recreates `res_dir` at every run (previous artifacts are wiped).

Pass `-r` to push results and `-p` to publish the HTML/XML/log artifacts (see
the `xtesting-run-publish` skill).

## Gotchas

- `dir` must be provided; missing it returns `EX_RUN_ERROR`.
- `options` must be a dict (the driver iterates its keys) — a list breaks it.
- The class-level counters (`Pytest.passed/failed/tests`) accumulate across
  repeated runs in the same process — restart to get a clean score.