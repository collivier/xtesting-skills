---
name: xtesting-ci-toolchain
description: Use when building a Xtesting test container image (Dockerfile) or deploying the Xtesting CI toolchain — Jenkins/GitLab jobs, results DB, S3 artifact repository — via the collivier.xtesting Ansible role (site.yml). This covers the "make world" and "play" sections of README.md: ansible-galaxy install, virtualenv, docker build/push to a local registry.
---

# Building the container and deploying the CI toolchain

Xtesting projects ship as Docker images run by CI. A full toolchain (test
scheduler/CI, results database, artifact repository) is deployed with the
`collivier.xtesting` Ansible role.

## Container image

A test project (driver + setup.py/setup.cfg + requirements.txt +
testcases.yaml) becomes a container:

```dockerfile
FROM alpine:3.24

ADD . /src/
RUN apk --no-cache add --update python3 py3-pip py3-wheel git py3-lxml && \
    git init /src && pip3 install --break-system-packages --no-cache-dir xtesting && \
    pip3 install --break-system-packages --no-cache-dir /src
COPY testcases.yaml /etc/xtesting/testcases.yaml
CMD ["run_tests", "-t", "all"]
```

Key points:
- Xtesting itself is installed directly in the image (`pip3 install xtesting`)
  so the container gets `run_tests` and the drivers without requiring the test
  project to declare it as a dependency.
- `pip3 install /src` builds via pbr (entry points land in the image).
- Alpine 3.20+ (incl. 3.24) marks Python as externally-managed (PEP 668) —
  pip installs must pass `--break-system-packages`, as in the example and in
  the official image.
- **pbr gotcha:** pbr computes the package contents with `git ls-files`, so a
  committed git checkout is required — `git init` alone (as in README) ships
  only `.py` files, silently dropping `testcases.yaml`, `.robot`, `.yml`,
  `logging.ini`… Add files to the index (`git add -A && git commit`) before
  building, or install from a released sdist/wheel. The official image build
  fetches a tagged git branch, which is why it carries all data files.
- `testcases.yaml` is copied to `/etc/xtesting/` so Xtesting picks it up
  (one of the search paths in `XTESTING_PATHES`).
- The default command runs every tier: `run_tests -t all`.
- Container keeps results in `/var/lib/xtesting/results`.
- Absolute `site-packages` paths in `testcases.yaml` must match the image's
  Python (e.g. `/usr/lib/python3.14/...` on Alpine 3.24, `/usr/lib/python3.12/
  ...` on the official image).

Build and publish to a local registry:

```bash
sudo docker build -t 127.0.0.1:5000/weather .
sudo docker push 127.0.0.1:5000/weather
```

## Deploying the toolchain ("make world")

`site.yml` drives the `collivier.xtesting` role:

```yaml
---
- hosts:
    - 127.0.0.1
  roles:
    - role: collivier.xtesting
      project: weather
      registry_deploy: true
      repo: 127.0.0.1
      dport: 5000
      suites:
        - container: weather
          tests:
            - humidity
            - pressure
            - temp
            - half
```

`suites.container` is the image pushed to the registry; `tests` are the
`case_name`s defined in `testcases.yaml`.

Deploy (from README.md):

```bash
virtualenv xtesting -p python3 --system-site-packages
. xtesting/bin/activate
pip install ansible
ansible-galaxy install collivier.xtesting
ansible-galaxy collection install ansible.posix community.general community.grafana \
    community.kubernetes community.docker community.postgresql
ansible-playbook site.yml
deactivate
rm -r xtesting
```

The role stands up the CI scheduler (Jenkins), the results DB, and the S3
artifact repository. The DB deployment runs MongoDB 5.0+ which requires the
_avx_ CPU instruction set — check with
`grep '^processor\|^flags.* avx' /proc/cpuinfo`. QEMU needs `-cpu max` (7.2+)
to expose avx.

## Playing with the deployed toolchain ("play")

- Jenkins: http://127.0.0.1:8080, login `admin` / password `admin`.
- The main job is `<project>-latest-daily` (e.g. `weather-latest-daily`),
  listed in the Jenkins view named after the project. Trigger a build with
  default parameters.
- The test case runs within seconds; open the
  `weather-127_0_0_1-weather-latest-<case>-run` job console to read:
  - the test output highlighting its status,
  - a link to the test DB entry for the results,
  - links to the published artifacts.
- `weather-latest-zip` prints the campaign zip produced by `zip_campaign`
  (DB dump + artifacts).

## Reference layout of a full project

From README.md "Write your own Xtesting driver":

```
weather.py       # driver class(es)
setup.py         # pbr packaging
setup.cfg        # [entry_points] xtesting.testcase = weather = weather:Weather
requirements.txt # xtesting, requests...
testcases.yaml   # tiers/testcases description
Dockerfile       # build + run_tests -t all
site.yml         # collivier.xtesting role invocation
```

See the `xtesting-write-driver`, `xtesting-testcases-config` and
`xtesting-run-publish` skills for each part.