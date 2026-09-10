---
name: xtesting-write-driver
description: Use when writing a custom Xtesting test case driver (a Python class registered as an xtesting.testcase entry point). This is the "Write your own Xtesting driver" workflow from README.md. Use when the user wants to add a new test case to a project based on xtesting, implement the run() method, register the driver in setup.cfg, or package a test project.
---

# Writing a custom Xtesting driver

Xtesting assembles sparse test cases: the developer only writes the test body
("driver") and a YAML description. Xtesting handles running, result scoring,
DB push, artifact publication and CI integration.

A driver is a Python class subclasses `xtesting.core.testcase.TestCase` and
registered as a stevedore driver under the `xtesting.testcase` namespace.

## The TestCase contract

A subclass must implement `run(self, **kwargs)` and set these instance
attributes so results can be scored and pushed:

| Attribute    | Meaning                                                    |
| ------------ | ---------------------------------------------------------- |
| `result`     | int score, 0-100. Compared against `criteria` from YAML.   |
| `start_time` | float epoch seconds, set at run start.                     |
| `stop_time`  | float epoch seconds, set at run end.                       |
| `details`    | optional dict published to the DB alongside the results.   |
| `res_dir`    | `RESULTS_DIR/<case_name>` = `/var/lib/xtesting/results/<case_name>`. Write dumps here to auto-publish them as artifacts. |

Exit codes (attributes on `TestCase`): `EX_OK` (0, PASS), `EX_RUN_ERROR`,
`EX_PUSH_TO_DB_ERROR`, `EX_TESTCASE_FAILED`, `EX_TESTCASE_SKIPPED`,
`EX_PUBLISH_ARTIFACTS_ERROR`.

`is_successful()` returns `EX_OK` when `result >= criteria` (criteria comes
from testcases.yaml, default 100). It can be overridden when scoring is not
suitable.

## Minimal driver (from README.md's weather example)

```python
#!/usr/bin/env python

import json
import os
import sys
import time

import requests

from xtesting.core import testcase


class Weather(testcase.TestCase):

    url = "https://samples.openweathermap.org/data/2.5/weather"
    city_name = "London,uk"
    app_key = "439d4b804bc8187953eb36d2a8c26a02"

    def run(self, **kwargs):
        try:
            self.start_time = time.time()
            req = requests.get("{}?q={}&&appid={}".format(
                self.url, self.city_name, self.app_key))
            req.raise_for_status()
            data = req.json()
            os.makedirs(self.res_dir, exist_ok=True)
            with open('{}/dump.txt'.format(self.res_dir), 'w+') as report:
                json.dump(data, report, indent=4, sort_keys=True)
            for key in kwargs:
                if data["main"][key] > kwargs[key]:
                    self.result = self.result + 100/len(kwargs)
            self.stop_time = time.time()
        except Exception:  # pylint: disable=broad-except
            print("Unexpected error:", sys.exc_info()[0])
            self.result = 0
            self.stop_time = time.time()
```

Note: `kwargs` received by `run()` are the `args` from the testcase's `run`
block in testcases.yaml. Catch every exception inside `run()`, never let one
escape (the runner logs a generic error otherwise).

## Other hooks a driver can override

- `check_requirements()`: set `self.is_skipped = True` to skip (e.g. when a
  binary is missing). See `xtesting.core.ansible.Ansible.check_requirements()`.
- `clean()`: delete resources after the run (called unless `-n`/`--noclean`).
- `push_to_db()`: publish results; see Xtesting's default implementation in
  `xtesting.core.testcase.TestCase`.
- `publish_artifacts()`: upload `xtesting.log`, `xtesting.debug.log` and all
  files under `res_dir` to S3. See default implementation for the required env
  vars (`S3_ENDPOINT_URL`, `S3_DST_URL`, `HTTP_DST_URL`).

## Registering the driver

Register the driver in `setup.cfg` under the `xtesting.testcase` entry point
(stevedore). The `run.name` in testcases.yaml must match the entry point name.

```
[entry_points]
xtesting.testcase =
    weather = weather:Weather
```

Also declare console scripts when packaging a full test project:

```
[entry_points]
console_scripts =
    run_tests = xtesting.ci.run_tests:main
    zip_campaign = xtesting.core.campaign:main
```

Use pbr-style packaging (setup.py with `pbr=True`, `setup_requires=['pbr>=2.0.0']`).

## Alternative base classes

- `xtesting.core.feature.Feature`: override `execute(self, **kwargs)` which
  must return 0 on success (anything else on failure). `run()` is provided.
- `xtesting.core.feature.BashFeature`: driver name `bashfeature`, runs any
  shell command via `args: cmd: <command>` in testcases.yaml. Supports
  `console` (stream to stdout), `max_duration` (seconds before kill) and
  `shell` (default false) args.
- `xtesting.core.vnf.VnfOnBoarding`: VNF test cases. Override `prepare()`,
  `deploy_orchestrator()` (optional), `deploy_vnf()` and `test_vnf()`; returns
  PASS only when all three steps succeed.

## Next steps

- Describe the test case in `testcases.yaml` (see the
  `xtesting-testcases-config` skill).
- Run it with `run_tests -t <case_name>` (see the `xtesting-run-publish` skill).
- Package it in a Docker image and deploy the CI toolchain (see the
  `xtesting-ci-toolchain` skill).