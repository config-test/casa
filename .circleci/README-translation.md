# CircleCI config: how it maps to the GitHub Actions workflows

`.circleci/config.yml` was derived from `.github/workflows/*.yml`, which are
treated as the source of truth for what CI does in this repo. Both systems can
run side by side; nothing in `.github/` was changed.

## Translated 1:1

| GitHub Actions workflow | CircleCI job |
|---|---|
| `rspec.yml` | `rspec` |
| `ruby_lint.yml` | `ruby_lint` |
| `erb_lint.yml` | `erb_lint` |
| `security.yml` (brakeman) | `brakeman` |
| `spec_checker.yml` | `spec_checker` |
| `factory_bot_lint.yml` | `factory_bot_lint` |
| `rake-after_party.yml` | `after_party` |
| `npm_lint_and_test.yml` | `npm_lint_and_test` |
| `yaml_lint.yml` | `yaml_lint` (approximate — see below) |

Commands, env vars and the Postgres 14.24 service were copied from the
workflows rather than re-derived. The RSpec job keeps the GitHub Actions
approach of one container running `parallel_test -n 4`, including the
`filesize` vs `runtime` grouping heuristic and the runtime-log cache that feeds
it. `.rspec_parallel` already writes JUnit XML to `tmp/reports/`, which
`store_test_results` consumes directly.

## Not translated 1:1

- **`yaml_lint.yml`** uses the `ibiqlik/action-yamllint` action. CircleCI has no
  equivalent, so the job installs the `yamllint` CLI and runs `yamllint .`,
  which picks up `.yamllint.yml` automatically. Behaviour should match the
  action's defaults, but it is a reimplementation, not a port.
- **Coverage upload (`qltysh/qlty-action`)** is *not* performed. The SimpleCov
  merge step still runs and coverage is stored as a build artifact, but no
  uploader is installed and no token is invented. To restore it, set
  `QLTY_COVERAGE_TOKEN` as a CircleCI project env var / context and add the
  qlty CLI invocation to the "Report coverage to qlty" step.
- **`codeql-analysis.yml`** is GitHub-native (CodeQL + the Security tab) and has
  no CircleCI equivalent. Leave it running on GitHub Actions.
- **`docker.yml`** runs the whole suite a second time through `docker compose`
  (including the `selenium/standalone-chromium` grid and bind-mounted volumes).
  On CircleCI this needs a `machine` executor, since `setup_remote_docker` does
  not support bind mounts. It was left out rather than guessed at; it duplicates
  the `rspec` job's coverage.
- **`combine_and_report.yml`** merges chunked test reports and uploads a runtime
  log to Azure Blob Storage. `store_test_results` replaces the reporting half
  natively; the Azure upload depends on `STORAGE_ACCESS_KEY` / `ACCOUNT_NAME` /
  `STORAGE_CONTAINER` secrets and was not recreated.
- **`after-deploy.yml`** and the repo-housekeeping workflows
  (`add-labels-based-on-column.yml`, `issue-auto-close-done.yml`,
  `issue-auto-unassign.yml`, `label.yml`, `remove-helped-wanted.yml`,
  `remove-label-based-on-column.yml`, `stale.yml`, `toc.yml`) drive the GitHub
  issue tracker and project board via `GITHUB_TOKEN`. They are not CI and should
  stay on GitHub Actions.
- **Trigger filters.** The workflows use `paths` / `paths-ignore` filters and
  only run on `main` plus PRs to `main` / `rfg-event-2025`. CircleCI 2.1 cannot
  express path filtering in a static config (it needs dynamic config with the
  path-filtering orb). The CircleCI workflow therefore runs every job on every
  push, which is a superset of the GitHub Actions behaviour.
- **`concurrency: cancel-in-progress`** (in `rspec.yml` and `docker.yml`) is a
  CircleCI *project setting* ("Auto-cancel redundant workflows"), not a config
  key. Enable it in the project UI.
- **`timeout-minutes`** per job has no direct per-job equivalent; CircleCI
  applies a global step timeout instead.
