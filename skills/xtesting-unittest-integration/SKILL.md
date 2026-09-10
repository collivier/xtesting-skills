---
name: xtesting-unittest-integration
description: Use when integrating your own Python unittest test suites (unittest.TestCase classes) into an Xtesting-based container — install the subunit tooling (subunit, subunit2junitxml, subunit2html) in the Dockerfile, reference importable module paths via the unit driver's name arg in testcases.yaml, and collect results.xml/results.html/subunit_stream.
---

# Running your own unittest suites in Xtesting

The default `unit` driver (already registered in Xtesting's `setup.cfg` as an
`xtesting.testcase` entry point) runs any `unittest` suite via
`unittest.TestLoader().loadTestsFromName()` and streams the output to **subunit**
for reporting — no custom Xtesting driver needed.

## 1. Make the test module importable in the image

`unit` loads tests **by Python dotted name** (`loadTestsFromName`), so your
`unittest.TestCase` classes must be importable inside the running container —
ship them in your pip-installed package. The Xtesting sample is
`xtesting.samples.fourth`:

```python
import unittest


class TestStringMethods(unittest.TestCase):

    def test_upper(self):
        self.assertEqual('Hello World'.upper(), 'HELLO WORLD')
```

> pbr note: `pip3 install /src` only ships non-`.py` files (fixtures, data)
> when the source tree is a git repository **with committed files** (`pbr`
> reads `git ls-files`). Your `.py` test modules always ship — but build from a
> committed checkout (or a released wheel) if your suites rely on external data
> files.

## 2. Install the dependencies in the Dockerfile

The driver needs the subunit tooling: `subunit-stats` and `subunit2junitxml`
(both shipped by the `python-subunit` PyPI package) and `subunit2html`
(shipped by `os-testr`). Installing `xtesting` already pulls in all three
(deprecated `subunit`/`subunit2junitxml` PyPI names and Alpine's
`py3-subunit*` packages are **not** required):

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git && \
    pip3 install --break-system-packages --no-cache-dir python-subunit os-testr && \
    pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

> Verify the three executables are on `$PATH`: `which subunit-stats
> subunit2junitxml subunit2html`.

## 3. Declare the test case in testcases.yaml

Driver name: `unit`. The `name` arg accepts anything
`TestLoader.loadTestsFromName()` understands: module, module.Class,
module.Class.method, or a discoverable package dir.

```yaml
tiers:
  - name: unit
    testcases:
      - case_name: my_unit_case
        project_name: myproject
        criteria: 100
        blocking: true
        run:
          name: unit
          args:
            name: mypkg.tests.test_foo
```

Packaged sample (case `fourth`):

```yaml
run:
  name: unit
  args:
    name: xtesting.samples.fourth
```

## 4. Scoring and results

The runner streams the suite through `SubunitTestRunner` and:

- Scores `result = 100 * (testsRun - failures - errors) / testsRun`
  (skips don't hurt, failures/errors do).
- `details = {testsRun, failures, errors}`, pushed to DB with `-r`.
- Generates `results.xml` (junit), `results.html`, and the raw
  `subunit_stream` under `res_dir` — all published with `-p`.
- Prints `subunit-stats` output to the xtesting log.

Run with `run_tests -t my_unit_case` (see the `xtesting-run-publish` skill).

## Gotchas

- `name` must resolve to an importable module or the suite is empty or
  `ImportError` → `EX_RUN_ERROR` / score 0 ("No test has been run").
- Without `name`, the driver runs the `suite` instance attribute of the
  subclass — a custom driver use case (see the `xtesting-write-driver` skill).
- The three subunit executables are required; if missing, report/timing
  generation fails.