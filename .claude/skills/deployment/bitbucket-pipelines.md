---
name: bitbucket-pipelines
description: Bitbucket Pipelines reference for the DevOps Engineer. Covers pipeline structure, branch strategies, Docker build/push workflows, caching, secrets, and deployment patterns.
---

# Bitbucket Pipelines

## File Location
`bitbucket-pipelines.yml` at the repository root.

## Pipeline Structure

```yaml
image: node:20-alpine   # default image for all steps unless overridden

definitions:
  caches:
    npm: ~/.npm
  services:
    docker:
      memory: 2048
  steps:
    - step: &lint-and-test
        name: Lint and Test
        caches:
          - npm
        script:
          - npm ci
          - npm run lint
          - npm run test
        artifacts:
          - coverage/**

    - step: &build-and-push
        name: Build and Push Docker Image
        services:
          - docker
        script:
          - IMAGE_TAG="${BITBUCKET_COMMIT:0:8}"
          - docker build -t $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG .
          - docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD $DOCKER_REGISTRY
          - docker push $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG
          - echo $IMAGE_TAG > image-tag.txt
        artifacts:
          - image-tag.txt

    - step: &deploy-staging
        name: Deploy to Staging
        deployment: staging
        script:
          - IMAGE_TAG=$(cat image-tag.txt)
          - pipe: atlassian/ssh-run:0.4.1
            variables:
              SSH_USER: $STAGING_SSH_USER
              SERVER: $STAGING_SERVER
              COMMAND: >
                docker pull $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG &&
                docker stop app-staging || true &&
                docker rm app-staging || true &&
                docker run -d --name app-staging
                  --env-file /etc/app/staging.env
                  -p 8080:8080
                  --restart unless-stopped
                  $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG

    - step: &deploy-production
        name: Deploy to Production
        deployment: production
        trigger: manual
        script:
          - IMAGE_TAG=$(cat image-tag.txt)
          - pipe: atlassian/ssh-run:0.4.1
            variables:
              SSH_USER: $PROD_SSH_USER
              SERVER: $PROD_SERVER
              COMMAND: >
                docker pull $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG &&
                docker stop app-production || true &&
                docker rm app-production || true &&
                docker run -d --name app-production
                  --env-file /etc/app/production.env
                  -p 8080:8080
                  --restart unless-stopped
                  $DOCKER_REGISTRY/$DOCKER_IMAGE_NAME:$IMAGE_TAG

pipelines:
  branches:
    'feature/*':
      - step: *lint-and-test

    develop:
      - step: *lint-and-test
      - step: *build-and-push
      - step: *deploy-staging

    main:
      - step: *lint-and-test
      - step: *build-and-push
      - step: *deploy-staging
      - step: *deploy-production   # manual trigger
```

---

## Branching Strategy

| Branch | Pipeline | Deployment |
|---|---|---|
| `feature/*` | Lint + test | None |
| `develop` | Lint + test + build + push | Staging (automatic) |
| `main` | Lint + test + build + push | Staging (auto) → Production (manual) |
| `hotfix/*` | Lint + test + build + push | Production (manual) |

---

## Secrets and Variables

**Never hardcode secrets in `bitbucket-pipelines.yml`.** Use Bitbucket repository or deployment variables.

### Variable types
- **Repository variables** — available to all pipelines in the repo. Use for: Docker registry credentials, shared API keys.
- **Deployment variables** — scoped to a deployment environment (staging, production). Use for: server addresses, environment-specific credentials.

### Required variables to configure in Bitbucket
Document all required variables in a `DEPLOYMENT.md` or in the pipeline file as comments:

```yaml
# Required Bitbucket repository variables:
# DOCKER_REGISTRY     — registry hostname (e.g., registry.hub.docker.com)
# DOCKER_IMAGE_NAME   — image name (e.g., myorg/myapp)
# DOCKER_USERNAME     — registry login username
# DOCKER_PASSWORD     — registry login password or token
#
# Required Bitbucket deployment variables (staging):
# STAGING_SERVER      — SSH hostname or IP of staging server
# STAGING_SSH_USER    — SSH user for staging deployment
#
# Required Bitbucket deployment variables (production):
# PROD_SERVER         — SSH hostname or IP of production server
# PROD_SSH_USER       — SSH user for production deployment
```

---

## Caching

Caches speed up pipelines by persisting downloaded dependencies between runs.

```yaml
definitions:
  caches:
    npm: ~/.npm           # Node.js
    pip: ~/.cache/pip     # Python
    gradle: ~/.gradle     # Java/Kotlin
    maven: ~/.m2/repository
```

Cache is keyed by branch. Caches are invalidated when the lock file changes.

---

## Docker Service Memory

Building Docker images in Bitbucket requires the Docker service. Default memory is 1024 MB — increase for large images:

```yaml
definitions:
  services:
    docker:
      memory: 2048
```

---

## Parallel Steps

Run independent steps in parallel to reduce total pipeline time:

```yaml
- parallel:
    - step:
        name: Unit Tests
        script: npm run test:unit
    - step:
        name: Integration Tests
        services: [database]
        script: npm run test:integration
```

---

## Deployment Verification

After a deployment step, verify the service is healthy before marking success:

```bash
# Wait for health check to pass (max 60 seconds)
for i in $(seq 1 12); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://$SERVER:$PORT/health)
  if [ "$STATUS" = "200" ]; then
    echo "Deployment healthy"
    exit 0
  fi
  echo "Waiting for health check... ($i/12)"
  sleep 5
done
echo "Health check failed after 60 seconds"
exit 1
```

---

## Common Pitfalls

- **Pipeline fails silently on SSH deploy** — always check exit codes from remote commands; use `set -e` in remote shell scripts
- **`:latest` deployed to production** — always use commit-based tags; `:latest` means you can't roll back to a specific version
- **Env file missing on server** — document in `DEPLOYMENT.md` that `/etc/app/<env>.env` must exist on the server before first deploy; the pipeline cannot create it
- **Docker login on every step** — login once in a shared step or use a wrapper; repeated logins can trigger rate limits
- **Large build context** — always have a `.dockerignore`; sending `node_modules` to the build daemon is slow and wastes time
