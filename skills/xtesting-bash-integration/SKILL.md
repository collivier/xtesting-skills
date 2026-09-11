---
name: xtesting-bash-integration
description: Use when integrating your own shell commands or scripts into an Xtesting-based container as test cases — declare bashfeature cases in testcases.yaml with the cmd arg (and console/max_duration/shell), no Python driver or extra framework dependency needed. Run scripts either inline via cmd or by copying the script into the image.
---

# Running your own bash commands/scripts in Xtesting

The default `bashfeature` driver (already registered in Xtesting's `setup.cfg`
as an `xtesting.testcase` entry point) executes any shell command as a test
case. It needs **no Python driver and no extra framework** — only the script or
command available inside the container.

## 1. Make the script available in the image

- **Inline command:** pass the whole command via `cmd` (see below) — nothing
  to copy.
- **Script file:** copy an executable script into the image and call it via
  `cmd`:
  ```dockerfile
  COPY mytest.sh /usr/local/bin/mytest.sh
  RUN chmod +x /usr/local/bin/mytest.sh
  ```
  or ship it inside your pip package and reference it by installed path.

`bashfeature` is bundled with Xtesting (`xtesting.core.feature.BashFeature`),
so no Python driver or extra framework is needed — install Xtesting itself in
the image (it provides `run_tests` and the driver) plus your package:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git && \
    pip3 install --break-system-packages --no-cache-dir xtesting && \
    git init /src && pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

## 2. Declare the test case in testcases.yaml

Driver name: `bashfeature`. Mandatory arg: `cmd`. Optional: `shell` (bool),
`console` (bool — stream output to stdout), `max_duration` (seconds — kills the
process on timeout).

```yaml
tiers:
  - name: bash
    testcases:
      - case_name: my_bash_case
        project_name: myproject
        criteria: 100
        blocking: true
        run:
          name: bashfeature
          args:
            cmd: /usr/local/bin/mytest.sh
            max_duration: 300
```

Packaged sample (case `third`):

```yaml
run:
  name: bashfeature
  args:
    cmd: echo -n Hello World; exit 0
    shell: true
```

## 3. Scoring and results

The driver runs `subprocess.Popen(cmd, shell=...)` and:

- `process.returncode == 0` → `result = 100`, else 0. **The result is
  boolean** — the command must exit 0 to PASS.
- Writes the full command output to `<res_dir>/<case_name>.log` (i.e.
  `/var/lib/xtesting/results/my_bash_case/my_bash_case.log`). This log is
  published with `-p`.
- `max_duration` exceeded → process killed → test FAILS (return code -2).

Run with `run_tests -t my_bash_case` and publish with `-r`/`-p` (see the
`xtesting-run-publish` skill).

## Gotchas

- `shell: true` runs through a shell — required for pipelines/`&&`/`;`
  commands (see the sample: `echo -n Hello World; exit 0` needs it).
- `shell: true` uses `/bin/sh`. If your scripts need bash features, install
  bash in the image (`apk add --no-cache bash`, as done by the official
  `opnfv/xtesting` image) and call `/bin/bash -c '...'` in `cmd`.
- With `shell: false` (default) the command must be a plain argv
  (`cmd: /usr/local/bin/mytest.sh`), not a shell string.
- Missing `cmd` returns `EX_RUN_ERROR`.
- `console: true` mirrors the command output into the xtesting log — handy for
  debugging, noisy otherwise.