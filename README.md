# Continuous Integration with GitHub Actions
 
Exercise project for setting up a Continuous Integration (CI) pipeline with GitHub Actions. The repository contains `app.py`, a minimal Python script created to have code to run through the pipeline (linting, and tests later on).
 
## CI Pipeline
 
The workflow is defined in `.github/workflows/ci.yml`.
 
**Trigger**: the pipeline runs automatically on every `push` to the repository — as soon as a commit is pushed, GitHub Actions starts the job.
 
**Jobs**: the workflow contains a single job, `lint`, running on an `ubuntu-latest` machine.
 
**Steps of the `lint` job**:
1. **Checkout** (`actions/checkout@v4`) — fetches the repository's code onto the runner, making it available to the following steps.

2. **Setup Python** (`actions/setup-python@v7`) — installs Python 3.13 on the runner.

3. **Run linter** — installs `flake8`, then runs it against `app.py` to check code style and catch obvious errors.

If `flake8` finds no issues, the job finishes successfully (green); otherwise, the run fails and the error details are visible in the logs.
 
## Successful run
 
Example of a run where the pipeline completed successfully: [run #32851346014](https://github.com/N-Haitu31/holbertonschool-continuous_integration/actions/runs/32851346014)
 
This link shows the `lint` job passing: checkout, Python setup, and linting of `app.py` all completed without errors.
 
## Running locally
 
```bash
pip install flake8
flake8 app.py
python app.py
```