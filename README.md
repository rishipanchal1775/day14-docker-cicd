# Day 14 — Docker CI/CD Pipeline with GitHub Actions

## 1. Project Overview

This project implements a basic **Continuous Integration and Continuous Delivery (CI/CD) pipeline** for a containerized Flask application using GitHub Actions, Docker, Docker Hub, Hadolint, and Trivy.

The pipeline automatically validates the application and Docker image when code is pushed to GitHub or a pull request is opened. For a push to the `main` branch, it publishes the image to Docker Hub only after the configured checks pass.

### Objectives

- Build a Docker image automatically using GitHub Actions.
- Lint the Dockerfile using Hadolint.
- Run a smoke test against the container's health endpoint.
- Scan the image for HIGH and CRITICAL vulnerabilities using Trivy.
- Publish successful images to Docker Hub.
- Tag images with the Git commit SHA for traceability.
- Test a published image outside the CI environment.
- Understand pipeline failures and basic rollback procedures.

## 2. Tools and Technologies

| Tool | Purpose |
|---|---|
| Docker | Packages and runs the application |
| Git and GitHub | Version control and source hosting |
| GitHub Actions | Automates the CI/CD pipeline |
| Docker Hub | Stores published container images |
| Hadolint | Checks Dockerfile best practices |
| Trivy | Scans container images for vulnerabilities |
| Flask | Provides the sample web application |
| `curl` | Tests HTTP endpoints |

## 3. Project Structure

```text
day14-docker-cicd/
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── README.md
└── .github/
    └── workflows/
        └── docker-ci.yml
```

**File responsibilities**

- `app.py`: Flask application with a home endpoint and health endpoint.
- `requirements.txt`: Python dependencies.
- `Dockerfile`: Defines how the application image is built.
- `.dockerignore`: Excludes unnecessary or sensitive files from the Docker build context.
- `.github/workflows/docker-ci.yml`: Defines the automated pipeline.
- `README.md`: Documents setup, execution, verification, troubleshooting, and rollback.

## 4. Application

### `requirements.txt`

```text
flask==3.0.3
```

Pinning the dependency version makes the intended Flask version explicit. It does not guarantee that every transitive dependency or the complete image is vulnerability-free.

### `app.py`

```python
import os
from flask import Flask, jsonify

app = Flask(__name__)


@app.get("/")
def home():
    return jsonify(
        message="Docker CI/CD lab",
        environment=os.getenv("APP_ENV", "dev")
    )


@app.get("/health")
def health():
    return "ok", 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**Endpoint explanation**

- `GET /` returns a JSON message and the configured application environment.
- `GET /health` returns HTTP 200 and the text `ok`.
- `APP_ENV` is read from an environment variable; if it is missing, the app uses `dev`.
- `0.0.0.0` allows the application to accept connections through the container's network interface.
- Port `5000` is the port on which Flask listens inside the container.

### `Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd --system --uid 10001 app
USER app

EXPOSE 5000

CMD ["python", "app.py"]
```

**Important instructions**

- `FROM`: Selects the base image.
- `ENV`: Sets Python runtime environment variables.
- `WORKDIR`: Sets the working directory inside the image.
- `COPY requirements.txt .`: Copies dependency definitions first, allowing Docker to reuse the dependency-installation layer when application code changes.
- `RUN pip install`: Installs the application dependency.
- `COPY app.py .`: Copies the application source.
- `RUN useradd`: Creates a dedicated application user.
- `USER app`: Runs the application as a non-root user.
- `EXPOSE 5000`: Documents the container's listening port; it does not publish that port to the host.
- `CMD`: Defines the default application command.

### `.dockerignore`

```text
.git
.github
.env
.venv
__pycache__/
*.pyc
reports/
```

This prevents matching files and directories from being sent as part of the Docker build context. It also avoids copying local Git history and virtual-environment files into the build context.

Do not put passwords, tokens, private keys, or other real credentials in the image or repository.

## 5. Build and Test Locally

Run these commands from the project directory.

### Step 1 — Build the image

```bash
docker build -t winter-arc-app:local .
```

Explanation:

- `docker build` builds an image from the Dockerfile.
- `-t winter-arc-app:local` gives the image a name and local development tag.
- `.` specifies the current directory as the build context.

### Step 2 — Check the image

```bash
docker images winter-arc-app
docker history winter-arc-app:local
docker image inspect winter-arc-app:local \
  --format '{{.Config.User}}'
```

The final command should report `app`, showing that the image specifies a non-root user.

### Step 3 — Start the container

```bash
docker run -d \
  --name local-api \
  -p 5000:5000 \
  winter-arc-app:local
```

`-d` runs the container in the background. `-p 5000:5000` maps host port 5000 to container port 5000.

### Step 4 — Test the endpoints

```bash
curl -f http://localhost:5000/health
curl -f http://localhost:5000/
docker logs local-api
docker ps
```

Expected health response:

```text
ok
```

The home endpoint should return JSON containing the message `Docker CI/CD lab` and the environment value.

### Step 5 — Remove the test container

```bash
docker rm -f local-api
```

This removes the container, not the image.

## 6. Configure Docker Hub and GitHub

### Step 1 — Create a Docker Hub repository

Create a repository named:

```text
winter-arc-app
```

Use your own Docker Hub username when referencing the image.

### Step 2 — Create a Docker Hub access token

Create an access token with only the permissions needed to publish the image. Use the token in GitHub Actions secrets instead of placing credentials in source code.

### Step 3 — Create the GitHub repository

Create a repository named:

```text
day14-docker-cicd
```

Push this local project to that repository.

### Step 4 — Add GitHub Actions secrets

In your GitHub repository, open **Settings → Secrets and variables → Actions**, and add:

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Your Docker Hub access token |

The workflow uses these secrets only for a push to the `main` branch. Pull requests do not need Docker Hub credentials to run the build, test, and scan stages.

**Never commit an access token to Git or paste it into the workflow file.**

## 7. GitHub Actions Workflow

Create `.github/workflows/docker-ci.yml` with the following content:

```yaml
name: docker-ci

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  build-test-scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image for test and scan
        uses: docker/build-push-action@v6
        with:
          context: .
          load: true
          tags: winter-arc-app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Smoke test
        run: |
          docker run -d \
            --name api \
            -p 5000:5000 \
            winter-arc-app:${{ github.sha }}

          for i in {1..10}; do
            if curl -fsS http://localhost:5000/health; then
              echo "Health check passed"
              docker rm -f api
              exit 0
            fi
            sleep 2
          done

          docker logs api
          docker rm -f api
          exit 1

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: winter-arc-app:${{ github.sha }}
          format: table
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: 1

      - name: Login to Docker Hub
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Tag image for Docker Hub
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        env:
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
        run: |
          docker tag \
            winter-arc-app:${{ github.sha }} \
            "$DOCKERHUB_USERNAME/winter-arc-app:${{ github.sha }}"

      - name: Publish commit-SHA image
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        env:
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
        run: |
          docker push \
            "$DOCKERHUB_USERNAME/winter-arc-app:${{ github.sha }}"
```

> **Maintenance note:** This workflow uses the action versions shown in this lab. Before using it in a production environment, review the actions' current releases and security guidance. For stronger supply-chain controls, consider pinning third-party actions to verified full commit SHAs.

## 8. Understand Each Pipeline Stage

### Stage 1 — Checkout source

```yaml
uses: actions/checkout@v4
```

Checks out the repository code on the GitHub-hosted runner.

### Stage 2 — Dockerfile linting

```yaml
uses: hadolint/hadolint-action@v3.1.0
```

Hadolint checks the Dockerfile for common mistakes and best-practice violations. A linting failure can stop later steps.

### Stage 3 — Buildx setup and image build

Buildx provides the builder used by the build action. The image is tagged with:

```text
winter-arc-app:<full-git-commit-sha>
```

`github.sha` is the commit identifier associated with the workflow event. It lets you connect the image tag to the source revision that produced it.

`load: true` loads the built image into the runner's Docker image store so the following smoke test and scan can use it.

### Stage 4 — Smoke test

The workflow starts a temporary container and repeatedly requests `/health`.

- If the endpoint responds successfully, the test passes and the temporary container is removed.
- If the endpoint does not become healthy within the retry window, the workflow prints container logs, removes the container, and fails.

A smoke test checks basic runtime behavior. It is not a substitute for a complete automated test suite.

### Stage 5 — Trivy image scan

Trivy scans the image for findings in the configured severities:

```text
HIGH,CRITICAL
```

The workflow uses `ignore-unfixed: true`, so findings without an available fix are excluded from this gate. `exit-code: 1` makes the step fail when matching findings remain.

If the scan fails, review the actual findings. Update the base image or dependencies where appropriate, rebuild the image, and scan again. Do not remove the gate just to make the workflow green.

A passing scan means the configured policy passed; it does not prove the image is completely secure.

### Stage 6 — Docker Hub login

This step runs only for a push to `main`. It authenticates using GitHub Actions secrets.

### Stage 7 — Tag and publish

The image is tagged with the Docker Hub username and full commit SHA, then pushed to Docker Hub.

The publish steps are conditional on a push to `main`. Pull requests can run the validation stages without publishing an image or receiving Docker Hub credentials.

## 9. Push the Project to GitHub

Initialize Git if this directory is not already a Git repository:

```bash
git init
git branch -M main
git add app.py requirements.txt Dockerfile .dockerignore \
  .github/workflows/docker-ci.yml README.md
git commit -m "Add Docker CI/CD pipeline"
```

If Git asks you to configure an identity, set your own name and email:

```bash
git config --global user.name "Your Name"
git config --global user.email "YOUR_GITHUB_EMAIL"
```

Connect your GitHub repository:

```bash
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/day14-docker-cicd.git
git push -u origin main
```

Replace the example username with your own. If `origin` already exists, inspect it with `git remote -v` instead of adding it a second time.

After pushing:

1. Open the repository on GitHub.
2. Select **Actions**.
3. Open the `docker-ci` workflow run.
4. Inspect each step and its logs.
5. Confirm the workflow succeeds.
6. Open Docker Hub and verify that the commit-SHA image tag exists.

## 10. Pull and Test the Published Image

Get the commit SHA:

```bash
git rev-parse HEAD
```

Copy the full SHA and use it as the image tag:

```bash
docker pull YOUR_DOCKERHUB_USERNAME/winter-arc-app:FULL_COMMIT_SHA
```

Replace both placeholders with the real Docker Hub username and full commit SHA.

Start the published image:

```bash
docker run -d \
  --name published-api \
  -p 5001:5000 \
  YOUR_DOCKERHUB_USERNAME/winter-arc-app:FULL_COMMIT_SHA
```

Test it:

```bash
curl -f http://localhost:5001/health
curl -f http://localhost:5001/
docker logs published-api
docker inspect published-api --format '{{.Config.Image}}'
```

Remove the container after testing:

```bash
docker rm -f published-api
```

Using host port `5001` avoids needing host port `5000` for this test. The image tag must already exist in Docker Hub; it will not exist if the main-branch pipeline has not successfully published it.

## 11. Safely Test a Failed Pipeline

This exercise checks that a broken application does not reach the publish steps. Use a separate branch, not `main`.

Create a test branch:

```bash
git switch -c test/broken-health
```

Temporarily introduce invalid Python syntax in `app.py`, then commit and push:

```bash
git add app.py
git commit -m "Test CI failure handling"
git push -u origin test/broken-health
```

Open the resulting pull request or inspect the workflow run for the branch. The build or test stage should fail before Docker Hub login and publishing.

**Important:** Do not merge intentionally broken code into `main`.

Restore the file from `main` on the test branch:

```bash
git restore --source=main -- app.py
git add app.py
git commit -m "Restore application code"
git push
```

Return to `main` when finished:

```bash
git switch main
git pull --ff-only origin main
```

This tests the build/test failure path. It does not independently prove the Trivy gate works; that requires observing the scan step reject an image with findings matching the configured policy.

## 12. Rollback Procedure

Commit-SHA tags help identify previously published images. If a newer release is problematic, identify an earlier known-good SHA and verify that its image tag still exists in Docker Hub.

Review recent commits:

```bash
git log --oneline -5
```

Pull the previous known-good image:

```bash
docker pull YOUR_DOCKERHUB_USERNAME/winter-arc-app:PREVIOUS_GOOD_SHA
```

Replace the placeholder with a real, previously published SHA.

Stop and remove an existing container with the same name, if one exists:

```bash
docker rm -f winter-app || true
```

Start the previous image:

```bash
docker run -d \
  --name winter-app \
  -p 5000:5000 \
  --restart unless-stopped \
  YOUR_DOCKERHUB_USERNAME/winter-arc-app:PREVIOUS_GOOD_SHA
```

Verify the rollback:

```bash
curl -f http://localhost:5000/health
docker logs winter-app
docker inspect winter-app --format '{{.Config.Image}}'
```

When finished with this local rollback test:

```bash
docker rm -f winter-app
```

This is a manual rollback demonstration, not a full production deployment strategy. Production rollback must also account for database changes, configuration compatibility, secrets, and any external services.

## 13. Troubleshooting

### Workflow does not appear on GitHub

Check that the file is committed at the exact path:

```text
.github/workflows/docker-ci.yml
```

Confirm that you pushed the branch and that the workflow's trigger matches the event.

### Hadolint fails

Read the Hadolint output and identify the reported rule. Fix the Dockerfile where appropriate and push a new commit. Do not ignore warnings without understanding them.

### The image build fails

Run the same build locally:

```bash
docker build --no-cache -t winter-arc-app:debug .
```

Check the failing Dockerfile instruction, dependency versions, and build logs.

### Smoke test fails

Check that Flask starts successfully and listens on `0.0.0.0:5000`. Inspect the local container logs:

```bash
docker logs local-api
```

Also confirm that the health endpoint returns HTTP 200.

### Trivy fails

Read the scan output to identify the affected package, installed version, severity, and available fix. Update the base image or dependency when appropriate, rebuild, and rerun the scan.

### Docker Hub authentication fails

Verify that `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` are configured as repository Actions secrets. Check the token's permissions and the target repository name. Never print the token in logs.

### Published image tag is missing

Confirm that the workflow ran for a push to `main`, passed all required checks, and reached the publish step. Copy the exact SHA from the successful run.

### Port 5000 is already in use

Inspect existing containers:

```bash
docker ps
docker ps -a
```

Remove only the old lab container that you have identified, or use a different host port, such as `-p 5001:5000`.

## 14. Evidence to Save for Submission

Capture real evidence from your own run:

- [ ] GitHub repository showing the project files.
- [ ] Successful GitHub Actions run with lint, build, smoke test, and Trivy stages.
- [ ] Docker Hub repository showing the published commit-SHA tag.
- [ ] Successful pull of the published image.
- [ ] Successful `/health` response from the pulled image.
- [ ] Evidence of a failed test run on a separate branch, if you complete the failure exercise.
- [ ] Rollback commands and the successful health check of the previous image.
- [ ] Notes describing any Trivy findings and how you addressed them.

Do not invent scan counts, image sizes, successful runs, or screenshots. Record the actual results from your environment.

## 15. Interview Questions and Answers

### Q1. What is CI/CD?

**Continuous Integration (CI)** automatically validates changes when code is pushed or a pull request is opened. **Continuous Delivery/Deployment (CD)** automates the steps needed to package and deliver validated software.

### Q2. What does GitHub Actions do in this project?

It runs the workflow jobs on a runner, checks out the source, lints the Dockerfile, builds the image, tests it, scans it, and publishes it on qualifying pushes to `main`.

### Q3. What is Hadolint?

Hadolint is a Dockerfile linter. It identifies common mistakes and recommends Dockerfile best practices.

### Q4. What is Trivy?

Trivy is a security scanner that can identify known vulnerabilities in container images and perform other security checks.

### Q5. Why run a smoke test?

It confirms that the built container can start and that a critical endpoint responds as expected before the image is published.

### Q6. What is the purpose of the `/health` endpoint?

It provides a simple endpoint that a test, monitoring system, or deployment process can request to check basic application availability.

### Q7. Why tag images with a Git commit SHA?

A commit-SHA tag connects an image to a specific source revision and makes it easier to identify and retrieve a previously built version. A tag is still mutable in registries unless additional controls are used; a digest is a stronger immutable content identifier.

### Q8. Why not use only the `latest` tag?

`latest` does not uniquely identify a source revision and can point to different images over time. Commit-SHA tags make the build's source revision clearer.

### Q9. Why are Docker Hub credentials stored in GitHub Actions secrets?

Secrets avoid placing credentials directly in the repository or workflow source. Access should be restricted to trusted workflows, and tokens should have the minimum necessary permissions.

### Q10. Why does publishing happen only on a push to `main`?

This workflow permits pull requests to run validation without publishing an image. Publishing is limited to the configured trusted branch after earlier steps succeed.

### Q11. What happens when a required workflow step fails?

The job stops at the failing step by default, so later publishing steps do not run.

### Q12. What does `exit-code: 1` do in the Trivy step?

It configures the action to return a failing status when the selected scan policy finds matching vulnerabilities. GitHub Actions then marks the step as failed.

### Q13. What does `ignore-unfixed: true` mean?

It excludes vulnerabilities for which Trivy reports no available fix from this scan gate. It does not mean those vulnerabilities are harmless.

### Q14. Why use a non-root user in the Docker image?

Running the application as a non-root user reduces the privileges available to the application process if it is compromised.

### Q15. What does `EXPOSE 5000` do?

It documents the port the containerized application listens on. Publishing a port to the host requires a runtime option such as `-p 5000:5000`.

### Q16. Why copy `requirements.txt` before `app.py`?

Dependency installation can be cached when the dependency file has not changed. Changes to application source then do not necessarily require reinstalling the dependencies.

### Q17. What is the difference between an image and a container?

An image is the packaged template used to start a container. A container is a running or stopped instance of that image with its own runtime state.

### Q18. What is a rollback?

A rollback returns an application to a previously known-good release. In this project, an earlier commit-SHA image can be pulled and run again.

### Q19. Does a successful Trivy scan guarantee that an image is secure?

No. It means the configured scan policy passed at that time. New vulnerabilities can be discovered later, and scanners cannot detect every possible security issue.

### Q20. How would you improve this pipeline for production?

Possible improvements include comprehensive automated tests, dependency and source-code scanning, verified action pinning, immutable image-digest deployment, least-privilege credentials, artifact provenance, protected branches, controlled deployments, and monitoring.

## 16. Final Checklist

- [ ] Flask application runs locally.
- [ ] Docker image builds successfully.
- [ ] Container runs as a non-root user.
- [ ] `/` and `/health` respond correctly.
- [ ] GitHub repository contains the workflow.
- [ ] Docker Hub username and token are configured as Actions secrets.
- [ ] Hadolint stage completes successfully.
- [ ] Image build completes successfully.
- [ ] Smoke test completes successfully.
- [ ] Trivy gate passes according to its configured policy.
- [ ] Successful `main` push publishes a commit-SHA image.
- [ ] Published image can be pulled and tested.
- [ ] Rollback steps are documented and understood.
- [ ] Evidence and actual scan findings are recorded.

## 17. Cleanup

Remove any test containers created during this lab:

```bash
docker rm -f local-api published-api winter-app 2>/dev/null || true
```

This command ignores errors if some containers do not exist. It does not remove your GitHub repository or Docker Hub images.

To remove the local image when you no longer need it:

```bash
docker image rm winter-arc-app:local
```

Keep any published image tags needed for rollback or submission evidence.

---

**Day 14 result:** A GitHub Actions pipeline that lints the Dockerfile, builds and smoke-tests a container image, scans it with Trivy, and publishes a commit-SHA-tagged image to Docker Hub only after the configured checks pass.
