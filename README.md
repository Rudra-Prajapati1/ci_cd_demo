# CI/CD Demo Project

A Python project demonstrating a production-grade CI/CD pipeline with GitHub Actions.

## Project Structure

```
ci_cd_demo/
├── src/
│   └── calculator.py              # Application logic
├── tests/
│   └── test_calculator.py         # 18 pytest test cases
├── .github/
│   ├── workflows/
│   │   ├── ci-cd.yml              # Main pipeline
│   │   └── reusable-test.yml      # Reusable test workflow
│   ├── actions/
│   │   └── setup-python-env/
│   │       └── action.yml         # Custom composite action
│   └── dependabot.yml             # Automatic action version updates
├── requirements.txt
├── pytest.ini
└── README.md
```

## Pipeline Overview

### Triggers

| Event | Branch | What runs |
|-------|--------|-----------|
| `push` | `dev` | tests → staging → production → auto-merge |
| `push` | `feature/**` | tests only |
| `pull_request` | `main`, `dev` | tests only |
| `workflow_dispatch` | any | manual trigger with reason input |
| `schedule` | — | daily at 3:55 PM IST |

### Jobs

#### `test-311` and `test-312`
- Calls the reusable workflow (`reusable-test.yml`) for Python 3.11 and 3.12 in parallel
- Uses a custom composite action for Python environment setup
- Runs flake8 lint check (max line length: 100)
- Runs pytest with coverage — fails if coverage drops below 80%
- Prints branch, commit SHA and actor on failure

#### `deploy-staging`
- Runs automatically after both test jobs pass
- Targets the `staging` environment
- Only triggers on push to `dev`

#### `deploy-production`
- Runs after staging deploy succeeds
- Targets the `production` environment
- Requires manual approval before running

#### `auto-merge`
- Runs after both test jobs pass
- Merges `dev → main` using `--no-ff`
- Uses `GITHUB_TOKEN` — no extra secrets needed
- `[skip ci]` in commit message prevents infinite loop

## Pipeline Flow

```
push to dev
      │
      ├── test-311 (Python 3.11) ──┐
      └── test-312 (Python 3.12) ──┘  both pass?
                    │
                    ├──→ deploy-staging (automatic)
                    │         │
                    │         ▼
                    │    deploy-production (requires approval ⏸️)
                    │
                    └──→ auto-merge dev → main ✅
```

## Reusable Workflow

`reusable-test.yml` is called by the main pipeline for each Python version. It accepts `python-version` as an input and returns `result` as an output. This avoids duplicating test logic across workflows.

## Custom Composite Action

`.github/actions/setup-python-env/action.yml` bundles three setup steps into one reusable action:
- Set up Python (via `actions/setup-python`)
- Restore pip cache (via `actions/cache`)
- Install dependencies from `requirements.txt`

Used in `reusable-test.yml` with:
```yaml
uses: ./.github/actions/setup-python-env
with:
  python-version: ${{ inputs.python-version }}
```

## Security

- All actions are pinned to exact commit SHA instead of version tags
- Default `GITHUB_TOKEN` permissions are set to `contents: read`
- Jobs that need elevated permissions declare them explicitly
- Dependabot runs weekly to open PRs for action version updates

## Dependabot

`.github/dependabot.yml` automatically checks for new action versions weekly and opens PRs with the updates. This keeps actions up to date without manual tracking.

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production branch — only updated by auto-merge bot |
| `dev` | Integration branch — triggers full pipeline on push |
| `feature/**` | Feature branches — runs tests only |

## Branch Protection Rules

Set up under: **Repo → Settings → Rules → Rulesets**

| Setting | Value |
|---------|-------|
| Target branch | `main` |
| Restrict deletions | ✅ Enabled |
| Block force pushes | ✅ Enabled |
| Require status checks | `Test Python 3.11`, `Test Python 3.12` |

## Run Locally

```bash
pip install -r requirements.txt
pytest
```

## Concurrency

```yaml
concurrency:
  group: ${{ github.ref }}
  cancel-in-progress: true
```

If two pushes happen quickly, the old pipeline run is cancelled and only the latest one runs.
