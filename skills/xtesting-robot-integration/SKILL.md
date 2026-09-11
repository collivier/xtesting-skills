---
name: xtesting-robot-integration
description: Use when integrating your own Robot Framework test files (.robot suites, resource files, libraries) into an Xtesting-based container — install robotframework in the Dockerfile, get the suites into the image, declare robotframework cases in testcases.yaml with suites/variable/deny_skipping args, and collect output.xml/report.html/log.html/xunit.xml results.
---

# Running your own Robot Framework tests in Xtesting

The default `robotframework` driver (already registered in Xtesting's
`setup.cfg` as an `xtesting.testcase` entry point) runs any Robot suites — no
custom Xtesting driver needed. You only provide your `.robot` files, the
`robotframework` dependency, and a `testcases.yaml` entry.

## 1. Make the suites available in the image

Your `.robot` files must be reachable by `robot.run()` inside the running
container:

- **pip package (recommended):** ship `.robot` files inside your Python
  package so they land under `site-packages` (this is how the
  `xtesting.samples.HelloWorld.robot` sample works). Reference them via the
  installed path — **match the image's Python version** (`/usr/lib/python3.12/
  ...` on the official image, `/usr/lib/python3.14/...` on Alpine 3.24…).
- **COPY layer:** copy them directly into the image:
  `COPY myrobot/ /tests/` then reference `/tests/`.

> pbr note: `pip3 install /src` only ships non-`.py` files (`.robot`,
> `resource.robot`) when the source tree is a git repo **with committed
> files** (`pbr` reads `git ls-files`). Build from a committed checkout or a
> released wheel, or the suite paths in `suites` won't resolve.

## 2. Install the dependency in the Dockerfile

`robotframework` is a dependency of xtesting — the `pip3 install xtesting`
line below pulls it in, so no explicit install is needed:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git && \
    pip3 install --break-system-packages --no-cache-dir xtesting && \
    git init /src && pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

## 3. Declare the test case in testcases.yaml

Driver name: `robotframework`. Mandatory arg: `suites` (list of file/dir
paths). Any other key becomes a `robot.run()` option (e.g. `variable`).

```yaml
tiers:
  - name: robot
    testcases:
      - case_name: my_robot_case
        project_name: myproject
        criteria: 100
        blocking: true
        run:
          name: robotframework
          args:
            suites:
              - /usr/lib/python3.14/site-packages/mypkg/tests/MySuite.robot
            variable:
              - 'var01:foo'
              - 'var02:bar'
            output: /var/lib/xtesting/results/my_robot_case/output.xml
```

Useful `robot.run()` args configurable here: `variable`, `variablefile`,
`include`, `exclude`, `tags`, `skip` (skip failing tests), `rerunfailed`…
See `robot.run` documentation — whatever is accepted there can be passed in
`args`.

## 4. Scoring and results

The driver runs `robot.run(*suites, **kwargs)` (log/report set to NONE, output
to `res_dir/output.xml`), then:

- `parse_results()` reads `output.xml` and computes
  `result = 100 * (passed + skipped) / total` by default,
  or `100 * passed / total` when `deny_skipping: true` (skips then count as
  failures).
- Generates `report.html`, `log.html` and `xunit.xml` into `res_dir`
  (`/var/lib/xtesting/results/<case_name>`).
- `details['tests']` holds per-test name/status/times; `details['description']`
  holds the suite name.

Publish these artifacts with `run_tests -t my_robot_case -p`, results to DB
with `-r` (see the `xtesting-run-publish` skill).

## Gotchas

- Variables must be passed as a list of `'NAME:value'` strings in `variable`.
- `suites` is popped from kwargs and is mandatory; missing it returns
  `EX_RUN_ERROR`.
- Libraries/`resource.robot` imported by your suites must also be reachable
  inside the image (same copy/package mechanism).