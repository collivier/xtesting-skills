# Xtesting Skills

Agent skills for working with the [Xtesting](https://xtesting.readthedocs.io) CI framework.
The `SKILL.md` format is identical for both Claude Code and opencode, so a single copy of
each skill is shared between the two ecosystems.

## Skills

| Skill | Purpose |
| --- | --- |
| `xtesting-write-driver` | Write a custom Xtesting test case driver (Python class, `run()` method, `setup.cfg` entry point). |
| `xtesting-testcases-config` | Author/edit `testcases.yaml` (tiers, run blocks selecting built-in drivers, criteria, dependencies). |
| `xtesting-run-publish` | Run test campaigns (`run_tests`), interpret results, publish artifacts / push to DB. |
| `xtesting-ci-toolchain` | Build the CI toolchain (Jenkins/GitLab, results DB, S3) and test container images via the Ansible role. |
| `xtesting-robot-integration` | Integrate Robot Framework suites into a test container. |
| `xtesting-behave-integration` | Integrate Behave BDD features into a test container. |
| `xtesting-pytest-integration` | Integrate Pytest test suites into a test container. |
| `xtesting-ansible-integration` | Integrate Ansible playbooks into a test container. |
| `xtesting-bash-integration` | Integrate shell commands/scripts into a test container. |
| `xtesting-unittest-integration` | Integrate Python `unittest` suites into a test container. |

All skills were verified end-to-end in Docker (`FROM alpine:3.24`, Python 3.14).

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json          # Claude Code marketplace manifest (skills listed explicitly)
├── skills/                       # the 10 skills, canonical location
│   └── xtesting-*/SKILL.md
├── opencode.example.json         # opencode config snippet consuming this repo
└── LICENSE
```

## Using with Claude Code

The repository is a plugin marketplace. Install the marketplace, then the plugin:

```
claude plugin marketplace add <owner-or-local-path>/xtesting-skills
claude plugin install xtesting
```

For a local clone, the marketplace can be added by path
(`claude plugin marketplace add /path/to/xtesting-skills`); once the repository is
published on GitHub, use `<owner>/xtesting-skills`.

The plugin skills are then available to the agent (invoked by their `description`).

## Using with opencode

Clone the repository and point `skills.paths` at the plugin skills directory. opencode
scans the path recursively for `SKILL.md`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/home/me/xtesting-skills/skills"]
  }
}
```

See `opencode.example.json`. Alternatively, symlink or copy each skill into
`~/.claude/skills/xtesting-*/SKILL.md` or `~/.agents/skills/xtesting-*/SKILL.md`
(opencode auto-loads external skills from those locations).

Both consumption methods expose the same skill content, since each `SKILL.md` carries
only the shared frontmatter (`name`, `description`) understood by both tools.

## License

Apache-2.0.