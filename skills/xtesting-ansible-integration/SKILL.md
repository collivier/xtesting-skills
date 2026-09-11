---
name: xtesting-ansible-integration
description: Use when integrating your own Ansible playbooks into an Xtesting-based container — install ansible-runner in the Dockerfile, get the playbook/private_data_dir into the image, declare ansible cases in testcases.yaml (private_data_dir, playbook, env), and collect runner stats as details. The result is boolean (100 on rc 0).
---

# Running your own Ansible playbooks in Xtesting

The default `ansible` driver (already registered in Xtesting's `setup.cfg` as
an `xtesting.testcase` entry point) runs any playbook via `ansible_runner` —
no custom Xtesting driver needed. It wraps `ansible_runner.run()` and only
needs a directory (the `private_data_dir`) with the playbook.

## 1. Make the playbook available in the image

`ansible` runs **inside a private data directory** (`private_data_dir`), which
must exist at run time. You must provide the directory with playbooks, roles,
and inventory:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git && \
    apk add --no-cache ansible openssh-client && \
    pip3 install --break-system-packages --no-cache-dir xtesting && \
    pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

> On Alpine, `ansible-playbook` comes from the `ansible` package (`apk add
> ansible`); on Debian/other images, install it via your package manager or
> `pip install ansible`. `ansible-runner` (the Python lib the driver uses) is
> already declared in xtesting's `requirements.txt`, so no extra pip install is
> needed once xtesting itself is installed (the `pip3 install xtesting` line
> above).
>
> Note: `check_requirements()` **skips the test case if `ansible-playbook` is
> not in `$PATH`**.
>
> pbr note: `pip3 install /src` only ships non-`.py` files (playbooks, roles,
> inventory) when the source tree is a git repo **with committed files** (`pbr`
> reads `git ls-files`). Build from a committed checkout or a released wheel,
> or your `private_data_dir` will be empty in the image.

## 2. Declare the test case in testcases.yaml

Driver name: `ansible`. Mandatory arg: `private_data_dir` (existing
directory). Then pass playbook name plus any `ansible_runner.run()` kwargs the
playbook needs (`inventory`, `playbook`, `envvars`, …). Extras `env` can set
container env vars for this test case only.

Packaged sample (`helloworld.yml`, case `eighth`, as shipped for the official
Python 3.12 image — adjust the path to your container's Python):

```yaml
run:
  name: ansible
  args:
    private_data_dir: /usr/lib/python3.12/site-packages/xtesting/samples
    playbook: helloworld.yml
```

Recipe for your own playbook copied under `/ansible/`:

```yaml
tiers:
  - name: ansible
    testcases:
      - case_name: my_playbook
        project_name: myproject
        criteria: 100
        blocking: true
        run:
          name: ansible
          args:
            private_data_dir: /ansible
            playbook: sitetest.yml
            inventory: hosts
```

## 3. Scoring and results

`ansible_runner.run(**kwargs)` is executed with `quiet=True` and
`artifact_dir = res_dir`. Then:

- `runner.rc == 0` → `result = 100`; otherwise `result` stays 0. **The
  playbook result is boolean — criteria is effectively 100 or 0**, whatever
  you set in testcases.yaml.
- `self.details = runner.stats` (task-level statistics), pushed to DB with `-r`.
- Runner artifacts/events are written under `res_dir`, so `-p` publishes the
  playbook run logs.

Launch with `run_tests -t my_playbook` (see the `xtesting-run-publish` skill).

## Gotchas

- Missing or non-directory `private_data_dir` returns `EX_RUN_ERROR`.
- `ansible-playbook` must be installed and on `$PATH`, or the case is SKIPped.
- The playbook often targets `127.0.0.1` (see `helloworld.yml`) — provide the
  right inventory/hosts for your target.