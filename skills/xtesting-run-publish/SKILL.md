---
name: xtesting-run-publish
description: Use when running Xtesting test campaigns, interpreting results, and publishing them — run_tests CLI flags (-t, -n, -r, -p), results directories, S3 artifact publishing (publish_artifacts), DB pushing (push_to_db), zip_campaign, and the required environment variables. Use when the user asks to run tests, check PASS/FAIL, or publish results/artifacts.
---

# Running tests and publishing results

## run_tests CLI

`run_tests` is the main entry point (`xtesting.ci.run_tests:main`). It parses
`testcases.yaml`, then runs, cleans, publishes artifacts and pushes results for
each test case.

```bash
run_tests -t <test_case_or_tier_or_all>
```

| Flag | Long        | Effect                                                    |
| ---- | ----------- | --------------------------------------------------------- |
| `-t` | `--test`    | Case name, tier name, or `all`. Defaults to `all` if omitted. |
| `-n` | `--noclean` | Do not clean resources after each test (default cleans).  |
| `-r` | `--report`  | Push results to the DB (default off).                     |
| `-p` | `--push`    | Push artifacts to the S3 repository (default off).        |

Exit value is 0 on success, negative (failed test/failure) otherwise. A failed
`blocking` test aborts its tier. `-t` also accepts an unknown name resolution
error: it logs "Unknown test case or tier" and returns error.

## Environment variables

Defaults are in `xtesting.utils.env.INPUTS` and can be overridden via the
environment or an env file sourced from `/var/lib/xtesting/conf/env_file`:

| Variable          | Default                                     | Purpose                  |
| ----------------- | ------------------------------------------- | ------------------------ |
| `CI_LOOP`         | `daily`                                     | e.g. daily/weekly        |
| `DEBUG`           | `false`                                     | `true` → debug logging from `logging.debug.ini` |
| `DEPLOY_SCENARIO` | `os-nosdn-nofeature-noha`                   | used to filter cases via `dependencies` |
| `INSTALLER_TYPE`  | `unknown`                                   |                       |
| `BUILD_TAG`       | (unset)                                     | version/run identifier; regex `daily|weekly-(.+?)-[0-9]*` extracts version |
| `NODE_NAME`       | (unset)                                     | pod/host name           |
| `TEST_DB_URL`     | `http://testresults.opnfv.org/test/api/v1/results` | results DB endpoint |
| `TEST_DB_EXT_URL` | –                                           | external URL shown to user |
| `S3_ENDPOINT_URL` | –                                           | e.g. `http://127.0.0.1:9000` |
| `S3_DST_URL`      | –                                           | e.g. `s3://xtesting/<prefix>` |
| `HTTP_DST_URL`    | –                                           | HTTP prefix for artifact links |

Env-file format: one `export KEY=value` per line (sourced by
`Runner.source_envfile`).

## Results

- Results dir: `/var/lib/xtesting/results` (`TestCase.dir_results`), one
  subdir per case (`<results>/<case_name>`).
- Logs: `xtesting.log` and `xtesting.debug.log` at the results dir root.
- Each case's result: PASS when `result >= criteria`, otherwise FAIL; SKIP when
  skipped via `check_requirements()`/`dependencies`/`enabled: false`.
- `run_tests` prints a per-case table and a final `Xtesting report` table across
  tiers.

## Publishing

### Results to DB

`push_to_db()` (called with `-r`): POST JSON to `TEST_DB_URL` containing
project/case name, installer, scenario, pod, build tag, criteria
(PASS/FAIL), start/stop dates, version (parsed from `BUILD_TAG`), and
`details`. On success it logs the result URL.

### Artifacts to S3

`publish_artifacts()` (called with `-p`): uploads `xtesting.log`,
`xtesting.debug.log`, and everything under the case's `res_dir` to the bucket
from `S3_DST_URL`, creating the bucket if missing. Requires S3 credentials
(`~/.aws/credentials`, `~/.boto`, or `AWS_ACCESS_KEY_ID` /
`AWS_SECRET_ACCESS_KEY`). It appends artifact links to `self.details["links"]`
and logs them.

### Zip a whole campaign

`zip_campaign` (`xtesting.core.campaign:main`, `Campaign.zip_campaign_files()`)
dumps all DB results matching `BUILD_TAG` plus all S3 artifacts into a
`<BUILD_TAG>.zip` in the S3 repository — used for third-party certifications.
Env vars required: `TEST_DB_URL`, `BUILD_TAG`, `S3_ENDPOINT_URL`, `S3_DST_URL`,
`HTTP_DST_URL`.

## Typical flow

```bash
run_tests -t my_tier -r -p          # run, report to DB, push artifacts
run_tests -t all                    # everything, no publication
run_tests -t my_case -n             # run one case, skip cleaning
```

## Debugging tips

- Set `DEBUG=true` for verbose logging.
- A case that crashes without catching errors logs:
  "Please fix the testcase <name>. All exceptions should be caught by the
  testcase instead!"
- Check `GET <TEST_DB_URL>?build_tag=<BUILD_TAG>` to verify pushed results.