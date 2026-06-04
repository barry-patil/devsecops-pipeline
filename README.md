# DevSecOps Pipeline

Security should be part of the pipeline, not a gate at the end. This is the CI/CD setup I've refined over a few projects — it runs security checks at every meaningful stage rather than doing a single scan before release and hoping for the best.

## Pipeline stages

```
Push / PR
  │
  ├── SAST + SCA          Bandit (Python static analysis) + Safety (dependency CVEs)
  │
  ├── Code Quality        SonarQube scan with quality gate enforcement
  │                       Fails the build if coverage < 80% or critical issues found
  │
  ├── Build + Scan        Docker multi-stage build → Trivy container scan
  │                       Fails on CRITICAL or HIGH CVEs in the final image
  │
  ├── Deploy (main)       kubectl rollout with 5-minute timeout
  │                       Slack alert on failure
  │
  └── DAST (PRs only)     ZAP baseline scan against staging URL
```

The DAST step only runs on pull requests, not on every main branch push — running a full ZAP scan on every deploy is too slow and the staging environment doesn't always have fresh data anyway.

## Why these tools

**Bandit** catches Python-specific issues like hardcoded passwords, use of `subprocess` with shell=True, and insecure deserialization. It's fast and produces very few false positives once you tune it.

**Trivy** for container scanning because it handles OS packages and language dependencies in one pass. Most other scanners need separate tools for each.

**SonarQube** for the stuff static analysis doesn't catch well — code smells, duplication, coverage trends over time. The quality gate enforcement is what makes it useful; without it, teams ignore the reports.

## Secrets required

Set these in your GitHub repository secrets:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM user with ECR push and EKS describe permissions |
| `AWS_SECRET_ACCESS_KEY` | Corresponding secret key |
| `ECR_REPOSITORY` | ECR repo name (not the full URI) |
| `SONAR_TOKEN` | SonarQube user token |
| `SONAR_HOST_URL` | Your SonarQube instance URL |
| `SLACK_WEBHOOK_URL` | Incoming webhook for deployment alerts |
| `STAGING_URL` | URL for DAST scanning on PRs |

## Running scans locally

```bash
# SAST
pip install bandit safety
bandit -r src/
safety check

# Container scan
docker build -t myapp .
trivy image myapp

# SonarQube (needs sonar-scanner installed)
sonar-scanner -Dsonar.projectKey=pratik-app
```

## Dockerfile notes

The Dockerfile uses a multi-stage build and runs as a non-root user. Both of these matter for Trivy scan results — the builder stage doesn't end up in the final image, and running as root is flagged as a high severity finding by most container scanners.

---

## Architecture

The full architecture diagram is in [architecture.drawio](./architecture.drawio). Open it at [app.diagrams.net](https://app.diagrams.net) — File → Open from Device → select the file.
