# Jenkins setup

## 1. Plugins

| Plugin | Needed for |
|---|---|
| Pipeline, Pipeline: Stage View | declarative pipeline, nested stages, `stash`, `input` |
| Git | checkout |
| Generic Webhook Trigger (2.x) | GitHub / Bitbucket webhook with a token credential |
| Credentials Binding, Plain Credentials | Secret text / file / username-password in steps |
| SonarQube Scanner for Jenkins | `withSonarQubeEnv`, `waitForQualityGate` |
| AWS Credentials | only if you use access keys instead of an IAM role (`AWS_CREDENTIALS_ID`) |
| Timestamper | timestamps in the log |
| Mailer | only for `NOTIFY_EMAIL` |
| PowerShell (built into Pipeline) | `powershell` step on the Windows agent |

## 2. Credentials (Manage Jenkins -> Credentials)

Only IDs appear in the Jenkinsfile. Secrets live here.

| Suggested ID | Type | CFG key | Used for |
|---|---|---|---|
| `git-credentials` | Username + token, or SSH key | `GIT_CREDENTIALS_ID` | checkout (GitHub PAT / Bitbucket app password) |
| `webhook-token` | Secret text | `WEBHOOK_TOKEN_CRED_ID` | webhook authentication |
| SonarQube token | Secret text, configured in the SonarQube server entry (`sonarqube-token`) | `SONAR_SERVER_NAME` | analysis upload |
| `aws-credentials` | AWS Credentials | `AWS_CREDENTIALS_ID` | ECR push/pull (leave empty to use the agent's IAM role - recommended) |
| `registry-credentials` | Username + password | `REGISTRY_CREDENTIALS_ID` | generic registry only |
| `nvd-api-key` | Secret text | `OWASP_NVD_API_KEY_CRED_ID` | fast OWASP database updates |
| `runtime-env-file` | Secret file | `DOCKER_ENV_FILE_CRED_ID` | runtime secrets for the container |
| `health-check-credentials` | Username + password | `HEALTH_CRED_ID` | health endpoint with basic auth |
| `slack-webhook-url`, `teams-webhook-url` | Secret text | `NOTIFY_SLACK_CRED_ID`, `NOTIFY_TEAMS_CRED_ID` | notifications |

Scope the credentials to the folder of the application where possible (least privilege) and use different credentials for production.

## 3. System configuration

* **SonarQube server:** Manage Jenkins -> System -> *SonarQube servers* -> name = `SONAR_SERVER_NAME`, URL, server authentication token. See `sonarqube-setup.md`.
* **SonarQube Scanner tool** (non-.NET apps, optional): Manage Jenkins -> Tools -> *SonarQube Scanner*; put the name in `SONAR_SCANNER_TOOL`.
* **Email** (optional): Manage Jenkins -> System -> *E-mail Notification* (SMTP server, port, TLS, credentials) and *System Admin e-mail address*. Then set `NOTIFY_EMAIL` (and optionally `NOTIFY_EMAIL_CC`, `NOTIFY_EMAIL_FROM`, `NOTIFY_ON`) in `CFG`.

## 4. Agents

| Label | OS | Used by |
|---|---|---|
| `linux-docker` (`AGENT_BUILD`, `AGENT_DOCKER`) | Linux | CI, scans, Docker |
| `windows-iis` (`AGENT_IIS`) | Windows Server | IIS deployment |

Run Windows agents as a **service under an Administrator account** (see `iis-deployment.md`). Give the Jenkins user on Linux access to Docker (`usermod -aG docker jenkins`). Keep production deploy agents separate from build agents.

## 5. Create the job

1. *New Item -> Pipeline.*
2. *Pipeline* section: **Pipeline script from SCM** (Git, your application repository, script path `Jenkinsfile`) - the Jenkinsfile and `scripts/` live in your repository. Set `USE_JOB_SCM: true` in CFG if you want exactly that checkout to be built, otherwise `REPO_URL` / `BRANCH` are used.
3. Save and **run it once manually** (Build with Parameters). Jenkins only registers the webhook trigger and the parameters after the first run.
4. If Jenkins flags a method in *In-process Script Approval*, review and approve it (the pipeline uses `currentBuild.getBuildCauses` for the production guard).

## 6. Webhook

Endpoint (no polling needed):

```
https://<jenkins-host>/generic-webhook-trigger/invoke?token=<value of the webhook-token secret>
```

Use HTTPS; the token is part of the URL. The webhook user does **not** need repository admin rights on the Jenkins side; creating the webhook needs permission on the repository.

| Platform | Where | Settings |
|---|---|---|
| GitHub | Repo -> Settings -> Webhooks -> Add | Payload URL as above, content type `application/json`, "Just the push event" |
| Bitbucket Cloud | Repo settings -> Webhooks -> Add | URL as above, trigger *Repository push* |
| Bitbucket Server / DC | Repo settings -> Webhooks -> Create | URL as above, event *Repository: Push* |

Only pushes to `BRANCH` start a build; other branches, tags and ping events are ignored. Webhook builds use the parameter defaults: `DEFAULT_ENVIRONMENT`, the default `DEPLOY_TYPE`, `ACTION=deploy`, `CONFIRM_PRODUCTION=false`, so they can never reach production.

**Poll SCM as fallback:** set `POLL_SCM_CRON` (for example `'H H/6 * * *'`). Leave it empty when the webhook works.

## 7. Parameters

| Parameter | Meaning |
|---|---|
| `ACTION` | `deploy` (full pipeline), `rollback` (restore previous version, nothing is built) or `discover-iis` (read-only list of IIS sites / paths / pools) |
| `DEPLOY_TYPE` | `docker`, `iis` or `none` (CI + security only) |
| `ENVIRONMENT` | `dev`, `staging`, `production` |
| `RUN_TESTS`, `RUN_SONARQUBE`, `RUN_OWASP`, `RUN_TRIVY` | per-run switches; a stage runs only if the matching `ENABLE_*` is true **and** the parameter is ticked |
| `CONFIRM_PRODUCTION` | must be ticked for any production deploy / rollback |
