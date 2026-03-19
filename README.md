# platform-pipelines

> Shared reusable deployment workflows for all application repositories in this organisation.
> Owned and maintained by the Platform Engineering team.

---

## What this repo is

Every application repo in this org generates a thin `deploy.yml` (via the CI/CD Pipeline Agent) that calls one reusable workflow from here — `platform-deploy.yml`. All actual deployment logic lives in this repo. Application repos contain zero deployment logic of their own.

```
java-game-service/deploy.yml  ──┐
payment-service/deploy.yml    ──┤──▶  platform-pipelines/platform-deploy.yml
user-service/deploy.yml       ──┘
... (100+ repos)
```

One change here = all repos get it instantly on their next deploy. No PRs to individual repos required.

---

## Repository structure

```
platform-pipelines/
└── .github/
    └── workflows/
        └── platform-deploy.yml   ← the only workflow file (for now)
```

---

## How it works

### 1. App repo `ci.yml` builds the image

The CI/CD Pipeline Agent generates `ci.yml` for every application repo. On every push to a deploy branch (`develop`, `release/*`, `staging`, `main`), `ci.yml` runs the full pipeline — tests, security scans, Docker build — and pushes the image to GCP Artifact Registry tagged with `github.sha`.

### 2. App repo `deploy.yml` calls this repo

When `ci.yml` succeeds, `deploy.yml` triggers and calls `platform-deploy.yml` here, passing three inputs:

```yaml
uses: YourOrgName/platform-pipelines/.github/workflows/platform-deploy.yml@main
with:
  image_tag:   ${{ github.event.workflow_run.head_sha }}
  repo_name:   ${{ github.event.repository.name }}
  environment: production   # or dev / test / staging
secrets: inherit
```

### 3. `platform-deploy.yml` handles everything

- Validates inputs
- Enforces GitHub environment approval gate
- Checks out the app repo
- Verifies the image exists in Artifact Registry
- Updates `helm/values-{environment}.yaml` with the new image tag
- Commits and pushes (ArgoCD detects and syncs to GKE)
- Runs smoke tests against the environment URL
- Sends Slack notification on success or failure

---

## Branch to environment mapping

| Git branch   | Environment  | Approval required      | Auto-deploy |
|--------------|-------------|------------------------|-------------|
| `develop`    | `dev`        | None                   | Yes         |
| `release/*`  | `test`       | None                   | Yes         |
| `staging`    | `staging`    | 1 QA approval          | After approval |
| `main`       | `production` | 2 approvals            | After approval |

Approval reviewers are configured in **GitHub Settings → Environments** — not in YAML.

---

## The golden rule

> Build the Docker image **once** in `ci.yml`. Every environment promotes the same SHA.  
> What QA approves in staging is exactly what runs in production. No rebuilds. No surprises.

---

## Secrets required in every app repo

Each application repo must have these GitHub secrets configured:

| Secret name        | Description |
|--------------------|-------------|
| `GCP_SA_KEY`       | GCP service account JSON for Artifact Registry access |
| `GCP_GITHUB_TOKEN` | GitHub PAT with `repo` write scope (platform needs this to commit helm values back to the app repo) |
| `SLACK_WEBHOOK_URL`| Slack incoming webhook for deploy notifications |
| `DEV_URL`          | Base URL of dev environment (e.g. `https://myapp-dev.example.com`) |
| `TEST_URL`         | Base URL of test environment |
| `STAGING_URL`      | Base URL of staging environment |
| `PROD_URL`         | Base URL of production environment |

> `GCP_GITHUB_TOKEN` is the most commonly missed secret. Without it, `platform-deploy.yml` cannot commit the updated helm values back to the app repo.

---

## Helm values file convention

Each app repo must have one values file per environment it deploys to:

```
helm/
  values.yaml            ← shared defaults (image name, port, resources)
  values-dev.yaml        ← dev overrides
  values-test.yaml       ← test overrides
  values-staging.yaml    ← staging overrides
  values-prod.yaml       ← production overrides
```

`platform-deploy.yml` updates only `image.tag` in the correct file. Everything else is left unchanged.

---

## Smoke test behaviour

After ArgoCD syncs the new image, `platform-deploy.yml` automatically runs a health check. It tries these endpoints in order until one returns `200`:

```
/health
/actuator/health   (Spring Boot)
/healthz           (Go)
/api/health
/
```

If none respond with `200`, the smoke test fails and blocks the next environment. If no URL secret is set for an environment, the smoke test is skipped with a warning.

---

## Adding a new workflow

If you need a new shared workflow (e.g. `platform-rollback.yml`, `platform-db-migrate.yml`), add it here under `.github/workflows/`. Application repos call it the same way — `uses: YourOrgName/platform-pipelines/.github/workflows/platform-rollback.yml@main`.

---

## Who to contact

For changes to deployment logic, smoke test behaviour, or approval gate configuration, raise a PR here and tag the **Platform Engineering** team for review.

Do not modify `platform-deploy.yml` without team review — all application repos depend on it.
