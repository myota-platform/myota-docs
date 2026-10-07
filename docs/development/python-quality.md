# Python style and quality checks

All Python code in the platform, service, deployment, and contracts
repositories is formatted with Ruff using a 79-column target and checked for
core PEP 8 errors, import problems, and unused or undefined names. A reusable
GitHub Actions workflow runs these checks on pushes and pull requests.

For local commit and push checks, install the development tools and hooks from
the root of a Python repository:

```sh
python3 -m pip install -r requirements-dev.txt
./scripts/install-quality-hooks.sh
```

The hook runs Ruff formatting and lint checks before commits and pushes. CI
repeats the checks so that bypassing a local hook does not bypass review-time
validation. To fix formatting and supported lint issues:

```sh
python3 -m ruff format .
python3 -m ruff check --fix .
```

Ruff's formatter targets 79 columns, but does not safely rewrite every long
SQL statement, URL, or semantic string literal. Where practical, wrap these
using adjacent literals without changing their value. The initial cleanup
formatted existing Python sources and removed detected unused imports,
variables, and lambda assignments.

Covered repositories are `myota-platform`, `myota-deploy`,
`myota-geodata-service`, `myota-activity-service`, `myota-identity-service`,
`myota-programme-service`, and `myota-contracts`. The canonical workflow and
organization-wide policy live in the [`.github` repository](https://github.com/myota-platform/.github).
