# CI/CD Demo Project

A Python project demonstrating a full CI/CD pipeline with GitHub Actions.

## Project Structure

```
ci-cd-demo/
├── src/
│   └── calculator.py        # Application logic
├── tests/
│   └── test_calculator.py   # Pytest test suite
├── .github/
│   └── workflows/
│       └── ci-cd.yml        # GitHub Actions pipeline
├── requirements.txt
├── pytest.ini
└── README.md
```

## CI/CD Pipeline

### Triggers
| Event | Branches | Jobs run |
|-------|----------|----------|
| `push` | `dev`, `feature/**` | test → auto-merge (dev only) |
| `pull_request` | `main`, `dev` | test |

### Jobs

#### 1. `test` — Runs on every push/PR
- Sets up Python 3.11 and 3.12 (matrix build)
- Installs dependencies with pip caching
- Runs `pytest` with coverage (must be ≥ 80%)
- Uploads coverage report as an artifact

#### 2. `auto-merge` — Runs only on push to `dev`
- Waits for all `test` jobs to pass (`needs: test`)
- Merges `dev → main` using `--no-ff` (preserves history)
- Uses `GITHUB_TOKEN` — no extra secrets needed

## Getting Started

### 1. Create the repo and branches
```bash
git init
git add .
git commit -m "initial commit"
git branch dev
git remote add origin https://github.com/YOUR_USERNAME/ci-cd-demo.git
git push -u origin main dev
```

### 2. Set branch protection on `main` (recommended)
Go to: **Settings → Branches → Add rule** for `main`:
- ✅ Require status checks to pass before merging
- ✅ Require branches to be up to date

### 3. Run locally
```bash
pip install -r requirements.txt
pytest
```

### 4. Trigger the pipeline
```bash
git checkout dev
# make a change ...
git add . && git commit -m "feat: my change"
git push origin dev
# → CI runs → if green, dev auto-merges into main
```

## How the Auto-Merge Works

```
push to dev
    │
    ▼
[test] job (Python 3.11 + 3.12)
    │  all pass?
    ▼
[auto-merge] job
    git checkout main
    git merge --no-ff dev
    git push origin main
```

The `[skip ci]` tag in the merge commit message prevents an infinite pipeline loop.