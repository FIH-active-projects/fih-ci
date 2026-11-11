# fih-ci

Shared checks for every Frugal Innovation Hub project. A project turns them on with two small
files, and every pull request (including pull requests from students' forks) is then checked for:

| Check | Tool | Blocks the merge? |
|---|---|---|
| **Secrets**: leaked keys and passwords, and key files such as `.env`, `*.pem`, `*.jks`, `serviceAccountKey.json` | [gitleaks](https://github.com/gitleaks/gitleaks) | Yes |
| **Clean repository**: no `node_modules/`, `build/`, `dist/`, `.dart_tool/`, virtual environments, or files over 10 MB | | Yes |
| **Code scan**: dangerous code such as `eval`, `shell=True`, SQL injection | [Semgrep](https://semgrep.dev) | Yes (only problems the pull request adds) |
| **Vulnerable libraries**: dependencies with known security bugs | [OSV-Scanner](https://github.com/google/osv-scanner) | No, it warns (can be switched on) |
| **Build and tests**: found automatically for Node.js (`package.json`), Python (`requirements.txt`, `pyproject.toml`) and Flutter (`pubspec.yaml`), in any folder | npm, pytest, flutter | Yes |
| **All checks passed**: one summary of the checks above | | Required check |
| **Protected files**: a pull request from a fork changes something under `.github/` | | Required check |

On pull requests, secrets, code and files are checked only in what the pull request adds, so old
problems in a project don't block students. Pushes to `main` check the whole project.

**Protected files** exists because GitHub runs a pull request's own version of the workflow files,
so a pull request could switch its checks off. That check runs from the project's `main` branch
instead, before anyone approves the pull request's workflows, and it never runs the pull request's
code. Pull requests from branches inside the project (FIH programmers, Dependabot) may change
`.github/`; pull requests from forks may not.

## Turn it on in a project

Add two files to the project. Use the organization that owns `fih-ci` in the `uses:` lines.

`.github/workflows/fih-checks.yml`:

```yaml
name: FIH checks

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: fih-checks-${{ github.ref }}
  cancel-in-progress: true

jobs:
  fih:
    name: FIH
    uses: FIH-active-projects/fih-ci/.github/workflows/checks.yml@v1
```

`.github/workflows/fih-guard.yml`:

```yaml
name: FIH guard

# pull_request_target runs this file from main, not from the pull request.
# Never add a step here that checks out or runs the pull request's code.
on:
  pull_request_target:
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: read

jobs:
  guard:
    name: Guard
    uses: FIH-active-projects/fih-ci/.github/workflows/guard.yml@v1
```

Then, in the project's branch rules for `main`, require the status checks **All checks passed**
and **Protected files**.

Also add a `.github/dependabot.yml` (see the test project for an example) so outdated
libraries get update pull requests.

### Options

```yaml
jobs:
  fih:
    name: FIH
    uses: FIH-active-projects/fih-ci/.github/workflows/checks.yml@v1
    with:
      block-on-vulnerable-libraries: true   # default false
      block-on-code-findings: false         # default true
      max-file-size-mb: 25                  # default 10
      allow-build-folders: true             # default false; for projects that commit their built site
      node-version: "20"                    # default "22"
      python-version: "3.11"                # default "3.12"
      flutter-version: "3.22.0"             # default: latest stable
```

### False alarms

- Code scan: add `# nosemgrep` (or `// nosemgrep`) at the end of the line and explain why in the pull request.
- Secrets: add a `.gitleaks.toml` with an allowlist to the project. Only do this for values that are
  truly not secret, such as test fixtures.

## Versions

Projects use the **`v1` branch** of this repo, not `main`. That way a mistake on `main` doesn't
break every FIH project at once.

1. Change the checks on `main`.
2. Try them on one project by temporarily changing its `@v1` to `@main`, and open a test pull request.
3. When they work, open a pull request from `main` into `v1` in this repo and merge it. Every
   project uses the new checks from its next run.

A change that would break existing projects goes to a new `v2` branch instead, and projects move to
`@v2` one by one.

This repo must stay **public** so that every FIH repo, public or private, can use it. It contains no secrets.
