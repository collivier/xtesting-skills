---
name: xtesting-testcases-config
description: Use when authoring, editing, or debugging a Xtesting testcases.yaml file. Covers the tiers/testcases schema, the run block that selects a built-in driver (bashfeature, robotframework, behaveframework, unit, ansible, pytest) and its args, criteria/blocking, dependencies filtering, and where Xtesting looks for the file.
---

# testcases.yaml

`testcases.yaml` is the single description of all test cases Xtesting will run.
It is loaded by `tier_builder.TierBuilder` and each test case executes a driver
picked via a stevedore entry point.

## Where Xtesting looks for it

Config files (`testcases.yaml`, `logging.ini`) are searched, in order, under:
`~/.xtesting`, `/etc/xtesting`, `<sys.prefix>/etc/xtesting` (see
`xtesting.utils.constants.XTESTING_PATHES`). The default packaged
`xtesting/ci/testcases.yaml` is used if none is found.

## Schema

Top level is a `tiers:` list, each tier grouping test cases:

```yaml
---
tiers:
  - name: simple
    order: 0
    description: ''
    testcases:
      - case_name: humidity
        project_name: weather
        criteria: 100
        blocking: true
        clean_flag: false
        description: ''
        run:
          name: weather
          args:
            humidity: 80
```

### Test case fields

| Field        | Required | Default | Meaning                                              |
| ------------ | -------- | ------- | ---------------------------------------------------- |
| `case_name`  | yes      |         | Unique test case name used with `run_tests -t`       |
| `project_name` | yes    |         | Project owning the test case                          |
| `criteria`   | no       | 100     | Score (0-100) that must be reached to PASS            |
| `blocking`   | no       | true    | A failure aborts the tier/campaign                    |
| `clean_flag` | no       | false   | Cleans resources after the run                        |
| `enabled`    | no       | true    | false moves the case to the skipped list              |
| `description`| no       | ''      | Human-readable description   |
| `dependencies` | no    |         | List of `{ENV_VAR: regex}`; case is skipped when the env var does not match the regex |
| `run`        | yes      |         | Selects the driver and passes args (see below)        |

### The run block

```yaml
run:
  name: <driver entry point>   # matches [entry_points] xtesting.testcase
  args: { ... }                # kwargs passed to driver run()
  env: { KEY: value }          # optional, set as env vars for this case only if unset
```

`args` are passed as `**kwargs` to the driver's `run(**kwargs)`. Optional `env`
sets environment variables for the test case (only if not already set).

### Built-in drivers and their args

| `run.name`       | Class                      | Relevant args                                        |
| -----------------| -------------------------- | ---------------------------------------------------- |
| `first`          | samples (hello world)      | –                                                    |
| `bashfeature`    | `Feature.BashFeature`      | `cmd`, `console` (bool), `max_duration` (s), `shell` (bool) |
| `robotframework` | `RobotFramework`           | `suites` (list, mandatory), `output`, `deny_skipping` (bool), plus any robot.run option |
| `behaveframework`| `BehaveFramework`          | `suites` (list of feature dirs, mandatory), `tags` (list), `console` (bool) |
| `unit`           | `Unit.Suite`               | `name` (TestLoader.loadTestsFromName(), e.g. `xtesting.samples.fourth`) |
| `ansible`        | `Ansible`                  | `private_data_dir` (mandatory dir), `playbook`, plus any ansible-runner kwarg |
| `pytest`         | `Pytest`                   | `dir` (mandatory path), `options` (dict or list of pytest options) |

Examples from `xtesting/ci/testcases.yaml`:

```yaml
run:
  name: bashfeature
  args:
    cmd: echo -n Hello World; exit 0
    shell: true
```

```yaml
run:
  name: robotframework
  args:
    suites:
      - /usr/lib/python3.12/site-packages/xtesting/samples/HelloWorld.robot
    variable:
      - 'var01:foo'
```

```yaml
run:
  name: ansible
  args:
    private_data_dir: /usr/lib/python3.12/site-packages/xtesting/samples
    playbook: helloworld.yml
```

```yaml
run:
  name: pytest
  args:
    dir: /usr/lib/python3.12/site-packages/xtesting/samples/fourth.py
```

### Dependencies (scenario filtering)

`dependencies` skip a test case when an environment variable does not match a
regex (used for scenario filtering):

```yaml
dependencies:
  - DEPLOY_SCENARIO: 'os-nosdn.*'
```

The case is skipped (listed with SKIP, not run) when `DEPLOY_SCENARIO` fails to
match. Multiple entries all must match.

## Validation checklist

- `tiers` is a list; each tier has a `name` and `testcases` list.
- Every test case has a unique `case_name` and a `project_name`.
- `run.name` matches an entry point declared in `setup.cfg` under
  `[entry_points] xtesting.testcase`.
- Mandatory driver args are present (e.g. `suites`, `dir`, `cmd`).
- `criteria` matches how the driver scores: hard 100/0 drivers (ansible, VNF)
  treat criteria as effectively boolean.