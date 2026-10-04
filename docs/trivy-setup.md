# Trivy setup

Trivy has two uses in this framework (`scripts/trivy-scan.sh`):

| Mode | Where | When |
|---|---|---|
| `image` | Docker deploy agent, after `docker build`, **before** the push | `DEPLOY_TYPE=docker` and `ENABLE_TRIVY` |
| `fs` | Linux build agent, after OWASP, result goes to the Security Gate | `TRIVY_FS_SCAN: 'auto'` = whenever the deployment is NOT a container (IIS / none); `true` = always; `false` = never |

A pure IIS deployment therefore never tries to scan a container image.

## Install
Linux agents: follow the official instructions (apt repository or install script) so that `trivy` is on the Jenkins user's PATH, and install `jq`. Alternative without installing: `TRIVY_USE_DOCKER: true` runs the `aquasec/trivy` container (needs Docker; pin a version with `TRIVY_DOCKER_TAG`). If trivy is missing the stage fails with instructions; it is never skipped silently.

## Policy
* `TRIVY_SEVERITY: 'HIGH,CRITICAL'` - which findings count.
* `TRIVY_EXIT_CODE: 1` - exit code when findings exist (`0` = report only).
* Image scan: findings go through `TRIVY_POLICY` immediately (`fail` stops before anything is pushed or deployed).
* Filesystem scan: result goes to the Security Gate.
* `TRIVY_SCANNERS: 'vuln,secret'` (add `misconfig` for Dockerfile / IaC), `TRIVY_IGNORE_UNFIXED`, `TRIVY_SKIP_DIRS`.

Reports: `reports/trivy/trivy-<mode>.json` and `.txt` (table), archived by Jenkins. Use a `.trivyignore` file in the repository for reviewed exceptions. Keep the vulnerability DB cache on the agent (`TRIVY_CACHE_DIR`) for speed.
