# Generic DevSecOps Jenkins Pipeline

**One Jenkinsfile, many applications, two deployment targets.** Configure an application by editing the `CFG` block of the `Jenkinsfile`; the pipeline then builds, tests, scans, gates and deploys it to **Docker (Linux, optionally via Amazon ECR)** or **.NET / IIS (Windows)** with backups, health checks and automatic rollback.

```
GitHub / Bitbucket → Jenkins → Checkout → Detect → Restore → Build → Test
  → SonarQube → OWASP Dependency-Check → Trivy → SECURITY GATE → (production approval)
  → DOCKER: build → image scan → registry → swap container ─┐
  → IIS   : publish → backup → deploy → recycle pool        ├→ Health check → SUCCESS
                                  failure → ROLLBACK ◄──────┘
```

## Contents
1. [Architecture](#1-architecture) · 2. [Pipeline flow](#2-pipeline-flow) · 3. [Prerequisites](#3-prerequisites) · 4. [Jenkins setup](#4-jenkins-setup) · 5. [Plugins](#5-required-jenkins-plugins) · 6. [Credentials](#6-jenkins-credentials) · 7. [SonarQube](#7-sonarqube-setup) · 8. [OWASP](#8-owasp-dependency-check-setup) · 9. [Trivy](#9-trivy-setup) · 10. [Docker](#10-docker-setup) · 11. [AWS / ECR](#11-awsecr-setup) · 12. [Windows / IIS agent](#12-windowsiis-agent-setup) · 13. [Webhooks](#13-webhook-setup) · 14. [Parameters](#14-pipeline-parameters) · 15. [Docker deployment](#15-docker-deployment) · 16. [IIS deployment](#16-iis-deployment) · 17. [Rollback](#17-rollback) · 18. [Health checks](#18-health-checks) · 19. [Troubleshooting](#19-troubleshooting) · 20. [Security best practices](#20-security-best-practices)

## Quick start
1. Copy `Jenkinsfile` and `scripts/` into your application repository (or keep `scripts/` in a central repo and set `SCRIPTS_REPO_URL`).
2. Edit **only** the `CFG` block. Placeholders (`YOUR_...`, empty required values) make the pipeline stop with a clear list of what is missing.
3. Create the credentials and agents ([4](#4-jenkins-setup)-[6](#6-jenkins-credentials)), create a *Pipeline script from SCM* job, run it once with **Build with Parameters**, then add the webhook ([13](#13-webhook-setup)).

Start on `dev` with `DEPLOY_TYPE=none` to validate CI and security stages first, then enable deployment.

## 1. Architecture
Details and diagrams: [docs/architecture.md](docs/architecture.md).

| Agent label (configurable) | Runs |
|---|---|
| `linux-docker` (`AGENT_BUILD`) | checkout, restore, build, test, SonarQube, OWASP, Trivy fs, security gate |
| `linux-docker` (`AGENT_DOCKER`) | docker build, Trivy image scan, registry push, container deploy, health check |
| `windows-iis` (`AGENT_IIS`) | dotnet publish, IIS backup, deploy, app pool recycle, health check |

Linux and Windows commands never share an agent; source travels with `stash`/`unstash`. The controller only runs `Initialize` and the approval prompt.

## 2. Pipeline flow
| # | Stage | Notes |
|---|---|---|
| 1 | Initialize | resolves config + environment overrides, validates, production protection |
| 2 | Checkout | GitHub/Bitbucket via Jenkins credentials, configurable branch, immutable tag `<build>-<sha>` |
| 3 | Detect Application Type | `dotnet` / `node` / `python` / `docker` / `generic` (or fixed by `APP_TYPE`) |
| 4 | Restore | `dotnet restore` / `npm ci` / `pip install -r requirements.txt` / custom `INSTALL_COMMAND` |
| 5 | Build | `dotnet build -c Release --no-restore` / `npm run build` / custom `BUILD_COMMAND` |
| 6 | Test | `dotnet test --no-build`; no test project = skipped gracefully, not failed |
| 7 | SonarQube | .NET: `begin -> build/test -> end`; others: `sonar-scanner`; optional Quality Gate |
| 8 | OWASP Dependency-Check | HTML/JSON/XML report archived, CVSS threshold configurable |
| 9 | Trivy | filesystem scan (non-container) / image scan (Docker) |
| 10 | **Security Gate** | SonarQube + OWASP + Trivy; secure default stops the pipeline |
| 11 | Production Approval | only for production |
| 12a | Docker | build -> image scan -> push -> deploy -> health check |
| 12b | IIS | publish -> backup -> deploy -> recycle -> health check |
| 13 | Rollback | automatic on failure, or `ACTION=rollback` |
| 14 | Notify / cleanup | console banner (✓ SUCCESS, ✗ FAILURE, ⚠ UNSTABLE), email / Slack / Teams, reports archived |

## 3. Prerequisites
* Jenkins 2.4xx+ with the plugins in [5](#5-required-jenkins-plugins)
* **Linux agent(s):** git, bash, curl, **jq**, Docker Engine (deploy agent), the toolchain of your apps (.NET SDK / Node / Python), Java 17+ (SonarScanner), OWASP Dependency-Check CLI, Trivy, AWS CLI v2 (ECR)
* **Windows agent:** IIS + WebAdministration, ASP.NET Core Hosting Bundle, .NET SDK, Git, Windows PowerShell 5.1, agent service running as an Administrator
* A SonarQube server (optional per app)

## 4. Jenkins setup
Step by step (system configuration, agents, job creation, script approval): [docs/jenkins-setup.md](docs/jenkins-setup.md).

## 5. Required Jenkins plugins
Pipeline (+ Stage View), Git, **Generic Webhook Trigger**, Credentials Binding / Plain Credentials, **SonarQube Scanner**, AWS Credentials (only for access-key auth), Timestamper, Mailer (only for email).

## 6. Jenkins credentials
Create them in *Manage Jenkins -> Credentials*; the Jenkinsfile only contains the **IDs**.

| ID (example) | Type | CFG key |
|---|---|---|
| `git-credentials` | Username + token / SSH key | `GIT_CREDENTIALS_ID` |
| `webhook-token` | Secret text | `WEBHOOK_TOKEN_CRED_ID` |
| SonarQube token | Secret text (inside the SonarQube server entry) | `SONAR_SERVER_NAME` |
| `aws-credentials` | AWS Credentials (or use an IAM role) | `AWS_CREDENTIALS_ID` |
| `registry-credentials` | Username + password | `REGISTRY_CREDENTIALS_ID` |
| `nvd-api-key` | Secret text | `OWASP_NVD_API_KEY_CRED_ID` |
| `runtime-env-file` | Secret file | `DOCKER_ENV_FILE_CRED_ID` |
| `health-check-credentials` | Username + password | `HEALTH_CRED_ID` |
| `slack-webhook-url` / `teams-webhook-url` | Secret text | `NOTIFY_SLACK_CRED_ID` / `NOTIFY_TEAMS_CRED_ID` |

## 7. SonarQube setup
[docs/sonarqube-setup.md](docs/sonarqube-setup.md): server token and webhook (`/sonarqube-webhook/`), Jenkins server entry, `dotnet-sonarscanner`, Quality Gate policy (`SONAR_POLICY`).

## 8. OWASP Dependency-Check setup
[docs/owasp-setup.md](docs/owasp-setup.md): install the CLI, NVD API key, persistent data folder, `OWASP_FAIL_THRESHOLD` (default 7).

## 9. Trivy setup
[docs/trivy-setup.md](docs/trivy-setup.md): install or run via Docker, `TRIVY_SEVERITY`, `TRIVY_EXIT_CODE`, `fs` vs `image` mode.

## 10. Docker setup
Docker Engine on the deploy agent, Jenkins user in the `docker` group. Details: [docs/docker-deployment.md](docs/docker-deployment.md).

## 11. AWS/ECR setup
Prefer the agent's **IAM role** (leave `AWS_CREDENTIALS_ID` empty). Needed: `REGISTRY_TYPE: 'ecr'`, `AWS_REGION`, `ECR_REPOSITORY`; the ECR repository (or `ECR_CREATE_REPO: true`). IAM permissions are listed in [docs/docker-deployment.md](docs/docker-deployment.md). The AWS account id is derived at runtime; nothing account-specific is stored in the repository.

## 12. Windows/IIS agent setup
[docs/iis-deployment.md](docs/iis-deployment.md): IIS features, Hosting Bundle, .NET SDK, agent service account, site and pool must exist.

## 13. Webhook setup
```
https://<jenkins>/generic-webhook-trigger/invoke?token=<webhook-token secret>
```
GitHub: *Settings -> Webhooks*, `application/json`, push events. Bitbucket Cloud: *Repository settings -> Webhooks*, trigger *Repository push*. Bitbucket Server/DC: *Repository settings -> Webhooks*, event *Push*. Only pushes to `BRANCH` build; webhook builds use `DEFAULT_ENVIRONMENT` and can never deploy to production. Optional low-frequency fallback: `POLL_SCM_CRON`. Run the job once manually first. Full steps: [docs/jenkins-setup.md](docs/jenkins-setup.md#6-webhook).

## 14. Pipeline parameters
| Parameter | Values / meaning |
|---|---|
| `ACTION` | `deploy` / `rollback` / `discover-iis` (read-only: lists IIS sites, paths and pools on the Windows agent) |
| `DEPLOY_TYPE` | `docker` / `iis` / `none` (CI + security only) |
| `ENVIRONMENT` | `dev` / `staging` / `production` |
| `RUN_TESTS`, `RUN_SONARQUBE`, `RUN_OWASP`, `RUN_TRIVY` | per-run switches (config `ENABLE_*` must also be true) |
| `CONFIRM_PRODUCTION` | required for any production deploy or rollback |

**Production protection:** `CONFIRM_PRODUCTION` + build started by a person (`PRODUCTION_REQUIRE_USER`) + branch in `PRODUCTION_BRANCHES` + manual approval (`PRODUCTION_APPROVAL`, `APPROVERS`, `APPROVAL_TIMEOUT_MIN`).

## 15. Docker deployment
Immutable tag -> `docker build` -> Trivy image scan (before anything leaves the agent) -> push (ECR / generic registry) -> save current version -> old container kept as `<name>-previous` -> new container -> health check -> finalize. Runtime secrets via env file, never in the image. See [docs/docker-deployment.md](docs/docker-deployment.md).

## 16. IIS deployment
**The deployment path is optional:** give `IIS_SITE_NAME` and the path + pool are read from IIS (or give only the path); run `ACTION=discover-iis` to see what exists on the server. Then: path safety checks, site/path/pool cross-check, timestamped backup, `app_offline.htm` (or stop **only** that pool), robocopy mirror with preserved folders, pool recycle, health check. See [docs/iis-deployment.md](docs/iis-deployment.md).

## 17. Rollback
* Docker: `<name>-previous` is restored (or the saved previous image is recreated).
* IIS: the backup taken for this deployment is restored, pool recycled.
* Triggered automatically when deploy / recycle / health check fails (`ROLLBACK_ENABLED`), or on demand with `ACTION=rollback`. The failed build always stays FAILED; the log states whether the rollback succeeded.

## 18. Health checks
`HEALTH_CHECK_URL`, `HEALTH_CHECK_RETRIES` (10), `HEALTH_CHECK_DELAY` (6 s), `HEALTH_CHECK_TIMEOUT`, `HEALTH_EXPECTED_STATUS`; optional basic auth via a Jenkins credential, TLS verification can be skipped only explicitly. Query strings are never printed.

## Example configurations

**Docker application on ECR (Linux)**
```groovy
APP_NAME: 'orders-api',            DEPLOY_TYPE: 'docker',
REPO_URL: 'https://github.com/<org>/orders-api.git',  BRANCH: 'main',
SONAR_PROJECT_KEY: 'orders-api',
REGISTRY_TYPE: 'ecr',              AWS_REGION: 'eu-west-1',   ECR_REPOSITORY: 'orders-api',
DOCKER_HOST_PORT: '8080',          DOCKER_CONTAINER_PORT: '8080',
HEALTH_CHECK_URL: 'http://localhost:8080/health',
ENV_OVERRIDES: [ production: [ AGENT_DOCKER: 'linux-docker-prod', DOCKER_HOST_PORT: '80' ] ]
```

**.NET application on IIS (Windows)**
```groovy
APP_NAME: 'billing-web',           DEPLOY_TYPE: 'iis',        APP_TYPE: 'dotnet',
REPO_URL: 'https://bitbucket.org/<ws>/billing-web.git',       BRANCH: 'main',
DOTNET_SOLUTION: 'src/Billing.sln', DOTNET_PROJECT: 'src/Billing.Web/Billing.Web.csproj',
IIS_SITE_NAME: 'Billing',          // path and pool are read from IIS (company did not give a path? this is enough)
// IIS_SITE_PATH: 'D:\\Sites\\Billing', IIS_ALLOWED_PATH_PREFIXES: 'D:\\Sites',   // only if you know the path
IIS_BACKUP_ROOT: 'D:\\Backups\\IIS', IIS_PRESERVE_PATHS: 'logs,uploads,appsettings.Production.json',
HEALTH_CHECK_URL: 'https://billing.example.com/health',
TRIVY_FS_SCAN: true
```

**CI-only application (no deployment yet):** `DEPLOY_TYPE: 'none'`.

## Email notifications
Set recipients in `CFG`; no plugin beyond Mailer is needed:

```groovy
NOTIFY_EMAIL: 'team@example.com, lead@example.com',   NOTIFY_EMAIL_CC: 'ops@example.com',
NOTIFY_ON: 'FAILURE,UNSTABLE',          // 'SUCCESS,FAILURE,UNSTABLE,ABORTED' = everything (default)
ENV_OVERRIDES: [ production: [ NOTIFY_EMAIL: 'prod-alerts@example.com', NOTIFY_ON: 'SUCCESS,FAILURE' ] ]
```
One-time: *Manage Jenkins -> System -> E-mail Notification* (SMTP server, port, TLS, credentials) and the system admin address. The HTML mail shows application, environment, branch/commit, build time, deploy target, health URL (no query string), the SonarQube / OWASP / Trivy results and links to the build, console log and reports. Subject: `[SUCCESS] my-app -> production (docker) #45`. A mail failure never fails the build; Slack / Teams work the same way through `NOTIFY_*_CRED_ID`.

## 19. Troubleshooting
| Symptom | Fix |
|---|---|
| `CONFIGURATION ERROR ...` | the listed CFG values are empty / placeholders |
| Webhook returns 200 but nothing builds | run the job once manually; token must match; pushed branch must equal `BRANCH` |
| `... could not run (tool missing or failed)` | install the tool on the agent (see the matching doc); exit code 10 is never a silent pass |
| Quality Gate times out | SonarQube -> Jenkins webhook missing |
| `SECURITY GATE FAILED` | fix the finding, or adjust the threshold / policy deliberately in CFG |
| `IIS site ... points to ...` | configured `IIS_SITE_PATH` / `IIS_APP_POOL` differ from IIS: fix them or leave them empty to read from IIS |
| Do not know the IIS site / path | run the job with `ACTION=discover-iis` |
| No email arrives | configure SMTP in Jenkins; check `NOTIFY_ON` and `NOTIFY_EMAIL`; the build log shows the mail result |
| `not a local Administrator` | run the Windows agent service as an administrator |
| ECR login / push denied | IAM role or credentials lack the ECR permissions listed in docs/docker-deployment.md |
| `docker: permission denied` | add the Jenkins user to the `docker` group, restart the agent |
| Health check fails after deploy | read the container / IIS logs printed by the rollback; check port, URL, connection strings |
| `No previous version is recorded` | first deployment (nothing to roll back to) |

## 20. Security best practices
* All secrets in Jenkins credentials; none in Git, scripts, Dockerfiles, env examples or logs (logins use stdin, URLs are hidden from `ps`, credentials are masked).
* Least privilege: IAM roles instead of keys, scoped credentials, separate production agents and credentials.
* Immutable image tags, vulnerability and secret scanning before anything is pushed, Quality Gate and dependency policy enforced by default.
* Production needs explicit confirmation, a human trigger, an allowed branch and an approval.
* Backup before every IIS deployment, rollback for both targets, health checks after every deployment.
* Safe by default: IIS path allow-list, only the configured pool is touched, preserved folders, `no-new-privileges` containers.
* Keep build and deploy agents separate; never expose the Docker socket over the network (use an agent on the target host).
* Review `SECURITY_GATE: warn/off` and every `*_POLICY: ignore` like code: they weaken the gate.

## Repository layout
```
generic-devsecops-pipeline/
├── Jenkinsfile                 # CFG block + pipeline
├── README.md
├── config/                     # docker.env.example, iis.env.example (variables the scripts read)
├── scripts/                    # bash (Linux) and PowerShell (Windows) implementation
│   └── lib/                    # shared helpers
└── docs/                       # architecture, Jenkins/SonarQube/OWASP/Trivy setup, Docker and IIS deployment
```

## Verification status (be aware before production use)
* **Tested by running:** all bash scripts passed `bash -n`; Docker deploy / rollback / finalize logic, health check (retries, status lists, auth, secret-safe logging), application-type detection and the missing-tool paths were exercised against fake `docker` / HTTP servers.
* **Reviewed, not executed:** the Jenkinsfile (Groovy) and the PowerShell scripts were checked structurally (bracket balance, escapes, unique stage names, variable names across files) but need a first run on real Jenkins / Windows. Pilot on `dev` with `DEPLOY_TYPE=none`, then `docker` / `iis` on a non-production host.
* Plugin option names (for example `tokenCredentialId` of the Generic Webhook Trigger) depend on plugin versions; if your Jenkins reports an unknown option, adjust that single line.
