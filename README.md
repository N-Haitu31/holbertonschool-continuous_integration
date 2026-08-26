# Continuous Integration with GitHub Actions
 
Exercise project for setting up a Continuous Integration (CI) pipeline with GitHub Actions. The repository contains `app.py` and `test_app.py`, run through the pipeline for linting and testing.
 
## CI Pipeline
 
The workflow is defined in `.github/workflows/ci.yml`.
 
**Trigger**: the pipeline runs automatically on every `push` to the repository, and on every `pull_request` targeting it when the PR is opened, synchronized (new commits pushed), or reopened — this lets both checks run on a PR before it is merged, not only after commits land directly on a branch.
 
**Jobs**: the workflow contains two jobs, `lint` and `test`, both running on an `ubuntu-latest` machine.
- `lint` — checks the code style of `app.py` with flake8.
- `test` — runs the test suite in `test_app.py` with pytest, using a matrix so it runs across several Python versions in parallel instead of just one.
**Steps of the `lint` job**:
1. **Checkout** (`actions/checkout@v4`) — fetches the repository's code onto the runner, making it available to the following steps.
2. **Setup Python** (`actions/setup-python@v7`) — installs Python 3.13 on the runner.
3. **Run linter** — installs `flake8`, then runs it against `app.py` to check code style and catch obvious errors.
**Steps of the `test` job**:
1. **Checkout** (`actions/checkout@v4`) — fetches the repository's code onto the runner, making it available to the following steps.
2. **Setup Python** (`actions/setup-python@v7`) — installs the Python version for that matrix instance (`3.11`, `3.12`, or `3.13`) on the runner.
3. **Run tests** — installs `pytest` and `flask`, then runs `pytest` against `test_app.py` on that Python version to check the app still behaves as expected.
If both jobs complete without errors, the pipeline finishes successfully (green); otherwise, the failing job's run shows the error details in its logs.
 
## Successful run
 
Example of a run where the pipeline completed successfully: [run #32945220553](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32945220553)
 
This link shows both the `lint` and `test` jobs passing together: checkout, Python setup, linting of `app.py`, and the `pytest` suite in `test_app.py` all completed without errors.
 
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
 
## Running locally
 
```bash
pip install flake8
flake8 app.py
python app.py
```
 
```bash
pip install pytest flask
pytest
```