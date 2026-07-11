# [FEATURE] Non-interactive flags for `apm init` project metadata (name, description, version, author)

**Is your feature request related to a problem? Please describe.**

`apm init`'s interactive mode collects four metadata fields — name, version,
description, author — but its non-interactive mode can freely set none of them,
so scripted / CI / `--yes` init is stuck with auto-detected boilerplate and must
hand-edit `apm.yml` afterward.

The command
([`init`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/init.py))
declares only these value-bearing options: the `project_name` argument, `--yes`,
`--plugin`, `--marketplace`, `--target`, `--verbose`. There is no
`--description`, `--version`, or `--author`. The two paths diverge on what a user
can control:

- Interactive
  ([`_interactive_project_setup`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/init.py))
  prompts for `name`, `version`, `description`, and `author`. In the current
  directory (no positional arg) it seeds the name prompt with the directory
  basename but lets the user type any name — a custom-named project in-place.
- Non-interactive (`--yes` or non-TTY stdin) takes
  [`_get_default_config`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/_helpers.py),
  which hardcodes `version: "1.0.0"`, `description: f"APM project for {name}"`,
  and `author` from `git config user.name`. No flag feeds `config`, so the field
  the interactive user is most likely to customize — the description — is the one
  a non-interactive user cannot set at all. For name, the positional arg is the
  only input, and `_perform_init` treats it as `mkdir(project_name)` + `chdir`
  into that subdir; a bare `.`/no-arg forces the current dir's basename. Neither
  path names a project in-place.

Current behavior — every metadata field is either rejected as an unknown option
or pinned to boilerplate:

```
$ apm init myproj --yes --description "My cool project"
Error: No such option: --description

$ apm init myproj --yes
$ cat myproj/apm.yml
name: myproj
version: 1.0.0
description: APM project for myproj      # boilerplate, unsettable
author: Jane Dev                         # git user.name, unsettable
```

**Describe the solution you'd like**

`--description`, `--version`, and `--author` flags on `apm init`, so every field
the interactive prompt collects can be set without the prompt:

```
apm init myproj --yes --description "My cool project" --version 0.1.0 --author "Jane Dev"
```

writes those exact values and exits 0. Also a way to name a project in the current
directory non-interactively (e.g. a `--name` flag decoupled from the positional
arg's mkdir behavior), matching what interactive init already allows in-place.

**Describe alternatives you've considered**

- Run `apm init` interactively, then re-run non-interactively — not viable in CI.
- Hand-edit or `sed` the generated `apm.yml` after `apm init --yes` — works but
  defeats the point of a scaffolding command and is brittle.

There is precedent in the CLI's own scaffolding surface: sibling
[`apm marketplace init`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/marketplace/init.py)
already takes `--name` and `--owner`, so "set metadata without the prompt" is an
established mode — just not on `apm init`.

**Additional context**

Observed on `apm` v0.24.0 (Linux, Python 3.14). The gap is visible from the
`apm init` flag surface alone (`apm init --help` lists no metadata flags) and the
two code paths cited above — no reproduction directory needed.
