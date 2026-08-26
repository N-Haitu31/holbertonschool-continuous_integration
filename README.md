# Continuous Integration with GitHub Actions
 
Exercise project for setting up a Continuous Integration (CI) pipeline with GitHub Actions. The repository contains `app.py` and `test_app.py`, run through the pipeline for linting and testing.
 
## CI Pipeline
 
The workflow is defined in `.github/workflows/ci.yml`.
 
**Trigger**: the pipeline runs automatically on every `push` to the repository, and on every `pull_request` targeting it when the PR is opened, synchronized (new commits pushed), or reopened — this lets both checks run on a PR before it is merged, not only after commits land directly on a branch.
 
**Jobs**: the workflow contains three jobs, `lint`, `test`, and `deploy`, all running on an `ubuntu-latest` machine.
- `lint` — checks the code style of `app.py` with flake8.
- `test` — runs the test suite in `test_app.py` with pytest, using a matrix so it runs across several Python versions in parallel instead of just one.
- `deploy` — represents the deployment step, gated so it only runs after `lint` and `test` have both succeeded, and only on the `main` branch.
**Steps of the `lint` job**:
1. **Checkout** (`actions/checkout@v4`) — fetches the repository's code onto the runner, making it available to the following steps.
2. **Setup Python** (`actions/setup-python@v7`) — installs Python 3.13 on the runner, restoring pip dependencies from cache when one matches instead of redownloading them.
3. **Run linter** — installs the dependencies from `requirements.txt`, then runs `flake8` against `app.py` to check code style and catch obvious errors.
**Steps of the `test` job**:
1. **Checkout** (`actions/checkout@v4`) — fetches the repository's code onto the runner, making it available to the following steps.
2. **Setup Python** (`actions/setup-python@v7`) — installs the Python version for that matrix instance (`3.11`, `3.12`, or `3.13`) on the runner, restoring pip dependencies from cache when one matches instead of redownloading them.
3. **Run tests** — installs the dependencies from `requirements.txt`, then runs `pytest` on that Python version to check the app still behaves as expected.
**Steps of the `deploy` job**:
1. **Use secret safely** — reads `DEPLOY_TOKEN` from the repository's secrets and only prints its length, never its value (see "Secrets and Gating" below).
If all three jobs complete without errors, the pipeline finishes successfully (green); otherwise, the failing job's run shows the error details in its logs.
 
## Successful run
 
Example of a run where the pipeline completed successfully: [run #33009267438](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/33009267438)
 
This link shows all three jobs passing together: `lint` and `test` complete first, then `deploy` runs after them on `main`, gated behind their success.
 
## Proof the `test` job actually checks the code
 
To confirm the `test` job reacts to the code and isn't just always green, a test assertion was deliberately broken on a PR, then fixed in a follow-up commit:
 
- Failing run (assertion broken): [run #32943398630](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32943398630)
- Passing run (assertion fixed): [run #32945043164](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32945043164)
## Matrix Testing
 
The `test` job runs on a **matrix** of 3 Python versions in parallel: `3.11`, `3.12`, and `3.13`.
 
Running the test suite on a single Python version can hide a bug that only shows up on another one — a syntax feature, a standard-library change, a dependency that behaves differently across versions. Testing all three at once, on every push and pull request, catches that kind of version-specific regression immediately instead of someone discovering it later on a different machine.
 
In the Actions tab, each version shows up as its own check with its own status — `test (3.11)`, `test (3.12)`, `test (3.13)` — grouped together under the `test` job.
 
Example of a run where the whole matrix passed: [run #32955433159](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32955433159)
 
This link shows all 4 checks green — `lint` plus the 3 `test` matrix versions — confirming the app behaves the same way across Python 3.11, 3.12, and 3.13.
 
## Caching
 
Both `lint` and `test` use the built-in pip cache of `actions/setup-python@v7` (the `cache: pip` input) — there is no separate `actions/cache` step; the caching happens inside the `setup python` step itself. When the cache key matches, pip's dependency cache is restored on the runner instead of being redownloaded from scratch.
 
To measure the gain, the same `run linter` step was timed on two runs of the `lint` job:
 
- **Without cache** (before caching was added): [run #10](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32955433159) — `run linter` took **6s**, job total **12s**.
- **With cache hit** (after caching was added, on a later push): [run #13](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32991820368/job/98250940437) — `run linter` took **4s**, job total **9s**. The `setup python` step log confirms `Cache hit for: setup-python-Linux-x64-...-pip-...` followed by `Cache restored successfully`.
That's roughly a **33% reduction** on the `run linter` step (and about 25% on the job as a whole) once the pip cache is warm.
 
## Secrets and Gating
 
### The secret
 
The `deploy` job uses one secret, `DEPLOY_TOKEN`. It's stored in the repository's own settings (**Settings → Secrets and variables → Actions**), not in `ci.yml`, and the workflow only ever references it through the `secrets` context: `${{ secrets.DEPLOY_TOKEN }}`, injected into the step as the `DEPLOY_TOKEN` environment variable. The value never appears anywhere in the YAML file itself.
 
To prove the token is never exposed, the step doesn't print `$DEPLOY_TOKEN` — it prints `${#DEPLOY_TOKEN}`, the bash syntax for a variable's **length**:
 
```bash
echo "Token length: ${#DEPLOY_TOKEN}"
```
 
That's a valid proof because a length is just a number (e.g. `Token length: 40`) — it confirms the secret was received by the job without ever putting the actual value in the logs. Even if this number were somehow sensitive, GitHub Actions also automatically masks any log output that matches a registered secret's value, replacing it with `***` — so the design here removes the risk entirely instead of relying on that safety net.
 
### The gating
 
The `deploy` job carries two conditions:
 
- **`needs: [lint, test]`** — `deploy` doesn't start until both `lint` and `test` have finished successfully. If either one fails, `deploy` is skipped automatically, GitHub Actions never launches it.
- **`if: github.ref == 'refs/heads/main'`** — even once `lint` and `test` pass, `deploy` only runs when the workflow is evaluating the `main` branch. It never runs for a `pull_request` or a push to any other branch.
Without `needs`, `deploy` would start in parallel with `lint` and `test`, regardless of whether they pass — meaning broken or unlinted code could reach the deploy step before anyone (or anything) knew it had failed. Without `if`, every push and every pull request — including ones from a feature branch or an external PR — would trigger a deploy attempt, which is both dangerous (deploying unreviewed or work-in-progress code) and pointless outside of `main`. Together, `needs` and `if` make sure `deploy` only ever runs once, on validated code, on the branch that's actually meant to ship.
 
### Proof
 
Example of a run where all three jobs ran in the right order: [run #33009267438](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/33009267438)
 
This run shows `lint` and `test` passing first, `deploy` starting only afterward on `main`, and its log printing the token's length without ever showing its value.
 
## Running locally
 
```bash
pip install -r requirements.txt
flake8 app.py
python app.py
```
 
```bash
pip install -r requirements.txt
pytest
```