# Docker deployment

`DEPLOY_TYPE=docker` runs on the Linux deploy agent (`AGENT_DOCKER`).

```
unstash → docker build → tag → Trivy image scan → registry push → save current version
        → stop old container (kept as <name>-previous) → run new container → health check
        → finalize (remove <name>-previous, prune old images)
        └ failure at any step after the swap → rollback → health check of the restored version
```

## Image tags
`<BUILD_NUMBER>-<short git sha>` (immutable). `latest` is only pushed when `DOCKER_PUSH_LATEST: true` (not recommended; ECR repositories with immutable tags reject it).

## Registry
| `REGISTRY_TYPE` | Settings |
|---|---|
| `ecr` | `AWS_REGION`, optional `ECR_REGISTRY` (empty = derived from the AWS account), `ECR_REPOSITORY` (default = image name), optional `ECR_CREATE_REPO`, credentials: the agent's **IAM role (recommended)** or `AWS_CREDENTIALS_ID` |
| `generic` | `REGISTRY_URL`, `REGISTRY_REPOSITORY`, `REGISTRY_CREDENTIALS_ID` (Username + password) |
| `none` | image stays on the agent that built it (build and deploy happen on the same agent) |

Logins use `--password-stdin`; passwords never appear on a command line or in the log.

### Minimum IAM permissions (ECR)
`ecr:GetAuthorizationToken` (resource `*`) and, on the repository: `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`, `ecr:DescribeRepositories`. Add `ecr:CreateRepository` only with `ECR_CREATE_REPO`. A deploy-only host needs just the pull permissions.

## Container settings
`DOCKER_CONTAINER`, `DOCKER_HOST_PORT` -> `DOCKER_CONTAINER_PORT`, `DOCKER_NETWORK`, `DOCKER_EXTRA_RUN_ARGS`, restart policy `unless-stopped`, `no-new-privileges` by default.
Runtime secrets: keep them **out of the image and out of Git**. Use `DOCKER_ENV_FILE` (a file that already exists on the host) or `DOCKER_ENV_FILE_CRED_ID` (Jenkins Secret file); the file is passed with `--env-file`.

## Rollback
* Before replacing: current image saved in `$DEPLOY_STATE_DIR/<APP_NAME>/<environment>/previous.env`; the old container is stopped and renamed `<name>-previous`.
* New container fails to start or the health check fails: the failed container is logged and removed, `<name>-previous` is renamed back and started, the health check runs again. The build stays FAILED.
* If the deployment failed before the old container was touched (e.g. image pull error) the rollback detects it and leaves the running version alone.
* `ACTION=rollback` restores the previous version at any time later (from `<name>-previous` if kept, otherwise by recreating it from the saved previous image).
* `ROLLBACK_ENABLED: false` disables the automatic rollback.

## Health check
`HEALTH_CHECK_URL` (default `http://localhost:<DOCKER_HOST_PORT>/`), `HEALTH_CHECK_RETRIES`, `HEALTH_CHECK_DELAY`, `HEALTH_CHECK_TIMEOUT`, `HEALTH_EXPECTED_STATUS`. Optional basic auth via `HEALTH_CRED_ID`.

## Deploying to another host
Run a Jenkins agent **on the target host** and give it its own label, then use `ENV_OVERRIDES` (`production: [AGENT_DOCKER: 'linux-docker-prod']`). The image reaches the host through the registry. This avoids exposing the Docker socket over the network.

## Cleanup
After a successful deployment old images of the repository are pruned (`DOCKER_KEEP_IMAGES`, default 5; running and previous images are always kept). Dangling images are pruned after the build (`DOCKER_PRUNE`). Nothing else is deleted.
