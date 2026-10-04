# SonarQube setup

## Server side
1. Create a project (or let the first analysis create it). Project key = `SONAR_PROJECT_KEY` (default `APP_NAME`).
2. Create a token for a CI user with *Execute Analysis* permission (a *Project Analysis Token* is enough). Store it **only** in Jenkins.
3. Add a webhook: *Administration -> Configuration -> Webhooks* -> `https://<jenkins>/sonarqube-webhook/`. Without it `waitForQualityGate` times out.

## Jenkins side
Manage Jenkins -> System -> *SonarQube servers*: name (`SONAR_SERVER_NAME`), URL, token credential (Secret text), "Environment variables" ticked. The pipeline never contains URLs or tokens; `withSonarQubeEnv` injects `SONAR_HOST_URL` and `SONAR_AUTH_TOKEN` for the analysis step only.

## .NET workflow (correct order)
The pipeline uses SonarScanner for .NET: `dotnet sonarscanner begin` -> `dotnet build` (+ `dotnet test`) -> `dotnet sonarscanner end`. For that reason, when SonarQube is on and the app is .NET, the Build and Test stages are replaced by **Build, Test and SonarQube (.NET)** (`scripts/sonar-dotnet.sh`). Requirements on the Linux build agent:

* .NET SDK, Java 17+ on PATH
* `dotnet tool install --global dotnet-sonarscanner` (once, for the Jenkins user), or `SONAR_DOTNET_AUTO_INSTALL: true`

Non-.NET apps use the generic `sonar-scanner` CLI (Jenkins tool `SONAR_SCANNER_TOOL`, or `sonar-scanner` on PATH).

## Quality Gate
`SONAR_QUALITY_GATE: true` waits for the result (`SONAR_QG_TIMEOUT_MIN`). Result `OK`/`WARN` = PASS, anything else = FAIL. A failed gate goes through `SONAR_POLICY`:

| Policy | Effect |
|---|---|
| `fail` (default) | Security Gate stops the pipeline, nothing is deployed |
| `unstable` | build becomes UNSTABLE, deployment continues |
| `ignore` | warning only |

`SECURITY_GATE: warn` downgrades every `fail` to `unstable`; `off` disables enforcement.

## Settings
`SONAR_PROJECT_KEY`, `SONAR_PROJECT_NAME`, `SONAR_AUTH_PROPERTY` (`sonar.login` for SonarQube < 10), `SONAR_EXTRA_ARGS` (.NET: `/d:sonar.exclusions=...`; generic: `-Dsonar.exclusions=...`).

## Troubleshooting
| Symptom | Fix |
|---|---|
| `dotnet-sonarscanner is not installed` | install the global tool or enable auto-install |
| Quality gate wait times out | the SonarQube -> Jenkins webhook is missing or blocked |
| `Unrecognized option sonar.token` | SonarQube < 10: set `SONAR_AUTH_PROPERTY: 'sonar.login'` |
| Analysis shows no coverage | add the coverage arguments for your tool through `SONAR_EXTRA_ARGS` |
