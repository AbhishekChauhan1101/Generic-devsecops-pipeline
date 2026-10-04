# Architecture

## Flow

```
GitHub / Bitbucket ──webhook──▶ Jenkins controller (schedules only)
                                      │
        ┌─────────────────────────────┴───────────────────────────────┐
        │  LINUX CI AGENT  (label: AGENT_BUILD, default linux-docker) │
        │  Checkout → Detect type → Restore → Build → Test            │
        │  → SonarQube → OWASP Dependency-Check → Trivy fs            │
        │  → SECURITY GATE → stash "source"                           │
        └─────────────────────────────┬───────────────────────────────┘
                                      │  production only: manual approval (no executor held)
                       ┌──────────────┴──────────────┐
                DEPLOY_TYPE=docker             DEPLOY_TYPE=iis
        ┌──────────────────────────────┐ ┌─────────────────────────────────┐
        │ LINUX DEPLOY AGENT           │ │ WINDOWS IIS AGENT               │
        │ (AGENT_DOCKER)               │ │ (AGENT_IIS, label windows-iis)  │
        │ unstash → docker build       │ │ unstash → dotnet publish        │
        │ → Trivy image scan           │ │ → backup current IIS app        │
        │ → registry push (ECR)        │ │ → deploy files → recycle pool   │
        │ → swap container             │ │ → health check                  │
        │ → health check               │ │ └ failure → iis-rollback.ps1    │
        │ └ failure → docker-rollback  │ └─────────────────────────────────┘
        └──────────────────────────────┘
                                      │
                       notifications (email / Slack / Teams) + archived reports
```

## Agent map

| Stage group | Agent | Needs |
|---|---|---|
| Initialize, Production Approval | none (controller, no executor) | nothing |
| CI + security scans | Linux, `AGENT_BUILD` | git, bash, jq, curl, the build toolchain (.NET SDK / Node / Python), Java 17 (Sonar), Dependency-Check, Trivy |
| Docker build / scan / push / deploy / health | Linux, `AGENT_DOCKER` | Docker Engine, Trivy, AWS CLI v2 (ECR), curl |
| dotnet publish, IIS backup / deploy / recycle / health | Windows, `AGENT_IIS` | .NET SDK, IIS + WebAdministration, Windows PowerShell 5.1, Administrator account |

Linux and Windows commands never run on the same agent. Source code travels between agents with `stash` / `unstash`.
`AGENT_DOCKER` and `AGENT_IIS` can be different per environment through `ENV_OVERRIDES` (for example a dedicated production agent that lives on the production host).

## Design decisions

* **One Jenkinsfile, configuration only in `CFG`.** Scripts contain no project values; they read environment variables that the Jenkinsfile exports from the effective configuration (`CFG` + overrides of the selected environment).
* **Scripts do the work, Jenkinsfile orchestrates.** Everything non-trivial is in `scripts/` so it can be linted (`bash -n`, ShellCheck, PowerShell parser) and run by hand.
* **Scans record results, the gate decides.** SonarQube, OWASP and Trivy-fs never abort the pipeline themselves (except when the tool is missing); each writes a result, the Security Gate evaluates all three together with a per-tool policy (`fail` / `unstable` / `ignore`) and a global mode (`enforce` / `warn` / `off`). Secure default: `enforce` + `fail`.
* **Tools are never silently skipped.** A missing tool = exit code 10 = the stage fails with setup instructions. To not use a tool, switch it off with `ENABLE_*=false`.
* **Immutable image tags** (`<BUILD_NUMBER>-<git sha>`); `latest` is off by default.
* **Rollback is built in.** Docker keeps the old container as `<name>-previous` until the new one is healthy; IIS takes a timestamped backup before touching anything. A failed deploy / recycle / health check triggers the rollback automatically (configurable) and the build stays FAILED so nobody mistakes it for a success.
* **Production is opt-in.** Production needs `CONFIRM_PRODUCTION`, a human-started build, an allowed branch and a manual approval. Webhook builds always use `DEFAULT_ENVIRONMENT`.

## Script exit codes

| Code | Meaning |
|---|---|
| 0 | success / policy satisfied |
| 1 | failure or policy violation (findings above threshold, health check failed, ...) |
| 2 | invalid configuration |
| 10 | tool missing or failed to run (setup problem, never treated as a pass) |

## Files

| Path | Purpose |
|---|---|
| `Jenkinsfile` | CFG block + pipeline |
| `scripts/ci-run.sh`, `detect-app-type.sh` | restore / build / test for dotnet, node, python, custom |
| `scripts/sonar-dotnet.sh`, `sonar-generic.sh` | SonarQube (begin → build/test → end for .NET) |
| `scripts/owasp-scan.sh`, `trivy-scan.sh` | dependency and vulnerability scans with explicit thresholds |
| `scripts/docker-*.sh` | build, push (ECR / generic), deploy, rollback |
| `scripts/health-check.sh` / `.ps1` | HTTP health check with retries |
| `scripts/dotnet-publish.ps1`, `iis-*.ps1` | publish, backup, deploy, recycle, rollback on Windows |
| `scripts/lib/` | shared helpers |
| `config/*.env.example` | variables the scripts read (for manual runs) |
