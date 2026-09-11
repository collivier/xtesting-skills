---
name: xtesting-behave-integration
description: Use when integrating your own Behave BDD test files (feature files, steps/) into an Xtesting-based container — install behave and behave-html-formatter in the Dockerfile, get the feature files into the image, declare behaveframework cases in testcases.yaml with suites/tags/console args, and collect output.json/output.html results.
---

# Running your own Behave tests in Xtesting

The default `behaveframework` driver (already registered in Xtesting's
`setup.cfg` as an `xtesting.testcase` entry point) runs any Behave feature
files — no custom Xtesting driver needed. You only provide your feature files
and step definitions, the `behave` dependency, and a `testcases.yaml` entry.

## 1. Make the feature files available in the image

Behave runs feature directories (with `steps/` inside). They must be reachable
inside the running container:

- **pip package (recommended):** ship the feature tree inside your Python
  package. The Xtesting sample lives at
  `xtesting/samples/features/` (see `xtesting/samples/features/hello.feature`
  and `steps/hello.py`) and is referenced in the packaged `testcases.yaml` via
  its site-packages path — **match the image's Python version**
  (`/usr/lib/python3.12/...` on the official image, `/usr/lib/python3.14/...`
  on Alpine 3.24).
- **COPY layer:** `COPY features/ /tests/features/` then reference
  `/tests/features/`.

Step definitions under `features/steps/` are discovered automatically by
Behave.

> pbr note: `pip3 install /src` only ships non-`.py` files (`.feature`,
> `steps/`) when the source tree is a git repo **with committed files** (`pbr`
> reads `git ls-files`). Build from a committed checkout or a released wheel,
> or the suite paths in `suites` won't resolve.

## 2. Install the dependencies in the Dockerfile

The driver is `behave` plus the HTML formatter plugin (`behave-html-formatter`,
used for `--format=behave_html_formatter:HTMLFormatter`). Both are dependencies
of xtesting — the `pip3 install xtesting` line below pulls them in, so no
explicit install is needed:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git py3-lxml && \
    pip3 install --break-system-packages --no-cache-dir xtesting && \
    pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

## 3. Declare the test case in testcases.yaml

Driver name: `behaveframework`. Mandatory arg: `suites` (list of feature
file/dir paths). Optional: `tags` (list), `console` (bool).

```yaml
tiers:
  - name: behave
    testcases:
      - case_name: my_behave_case
        project_name: myproject
        criteria: 100
        blocking: true
        run:
            name: behaveframework
            args:
              suites:
                - /usr/lib/python3.14/site-packages/mypkg/tests/features
              tags:
                - foo
```

From the packaged sample `testcases.yaml` (case `sixth`, as shipped for the
official Python 3.12 image — adjust the path to your container's Python):

```yaml
run:
  name: behaveframework
  args:
    suites:
      - /usr/lib/python3.12/site-packages/xtesting/samples/features
    tags:
      - foo
```

## 4. Scoring and results

The driver runs `behave` with `--junit`, JSON and HTML formatters, then:

- `parse_results()` reads `res_dir/output.json` and computes
  `result = 100 * passed / total` (skipped and failed lower the score).
- Writes `res_dir/output.json`, `res_dir/output.html`, and JUnit XML into
  `res_dir`.
- `details` include `total_tests`, `pass_tests`, `fail_tests`,
  `skip_tests`, `tests` (per-scenario statuses).

Artifacts: pass `-r` to push results to DB and `-p` to publish the JSON/HTML/
XML outputs (see the `xtesting-run-publish` skill).

## Gotchas

- `tags: [foo]` becomes `--tags=foo` — scenarios without the tag are skipped.
- `suites` is mandatory; missing it returns `EX_RUN_ERROR`.
- Step definition files must not be inside a non-discoverable path; keep them
  under `<features>/steps/`.
- Concretely, scenarios tagged with `@foo` are picked up by the packaged sample
  above (see `hello.feature`).