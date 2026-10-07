# Python style and quality checks

All Python code in the platform, service, deployment, and contracts
repositories is formatted with Ruff using a 79-column target and checked for
core PEP 8 errors, import problems, and unused or undefined names. A reusable
GitHub Actions workflow runs these checks on pushes and pull requests.

For local commit and push checks, install the development tools and hooks from
the root of each Python repository:

```sh
./scripts/install-quality-hooks.sh
```

The installer creates a repository-local `.venv-quality`, installs pinned
Ruff there, and configures Git to use the tracked `.githooks` scripts. They
run Ruff formatting and lint checks before commits and pushes. CI repeats the
checks so bypassing local hooks does not bypass review-time validation. To fix
formatting and supported lint issues:

```sh
.venv-quality/bin/ruff format .
.venv-quality/bin/ruff check --fix .
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
