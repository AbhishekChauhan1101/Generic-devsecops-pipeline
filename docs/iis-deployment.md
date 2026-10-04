# IIS deployment (.NET)

`DEPLOY_TYPE=iis` runs on the Windows agent (`AGENT_IIS`) using Windows PowerShell 5.1.

```
unstash → dotnet publish → backup current IIS app → deploy files → recycle app pool → health check
                                                       └ failure after the backup → iis-rollback.ps1 → health check
```

## Prerequisites on the Windows agent
* IIS with *Management Scripts and Tools* (WebAdministration module); the **ASP.NET Core Hosting Bundle** for your .NET version
* .NET SDK, Windows PowerShell 5.1, `robocopy` (built in)
* Jenkins agent installed as a **service running under a local Administrator account** (needed to manage the pool and write to the site folder)
* The IIS **site and application pool already exist**; the pipeline deploys files, it does not create sites
* Outbound access to NuGet / your package feed

## Where to deploy: the path is optional
The company does not always give you the physical path. You do not need it:

| You configure | What the pipeline does |
|---|---|
| `IIS_SITE_NAME` only (**simplest**) | reads the physical path **and** the application pool from IIS |
| `IIS_SITE_PATH` only | finds the site / application / pool that owns that path in IIS (stops if none or several match) |
| both | cross-checks them; a mismatch stops the pipeline (wrong-folder protection) |

`IIS_APP_POOL` is optional too; if you set it, it must match IIS. `IIS_ALLOWED_PATH_PREFIXES` is only required when you type `IIS_SITE_PATH` yourself (a path read from IIS is trusted, but drive roots, shared IIS roots and system folders stay blocked).

**Do not know the site name either?** Run the job with `ACTION = discover-iis`. It runs read-only on the Windows agent and prints every site, application, physical path, pool and binding of that server, plus a snippet to paste into `CFG`. The same script can be run by hand: `.\scripts\iis-discover.ps1`.

If the folder does not exist yet (brand-new site) set `IIS_CREATE_PATH: true` for the first deployment.

## Settings (CFG)
`DOTNET_PROJECT`, `IIS_SITE_NAME` and/or `IIS_SITE_PATH`, `IIS_APPLICATION` (optional), `IIS_APP_POOL` (optional), `IIS_ALLOWED_PATH_PREFIXES` (only with `IIS_SITE_PATH`), `IIS_BACKUP_ROOT`, `IIS_BACKUP_KEEP`, `IIS_PRESERVE_PATHS`, `IIS_STRATEGY`, `IIS_PURGE`, `IIS_CREATE_PATH`, `IIS_REQUIRE_WEB_CONFIG`, `IIS_GRANT_PERMISSIONS`, `IIS_WRITABLE_PATHS`, `HEALTH_CHECK_URL`.
Windows paths in `CFG` need doubled backslashes (`'D:\\Sites\\MyApp'`) or forward slashes.

## Safety checks (all must pass before anything changes)
1. The agent account is a local Administrator.
2. The deployment path (configured or read from IIS) is a local drive path, at least two levels deep, **not** `C:\inetpub`, `C:\inetpub\wwwroot`, a drive root or a Windows / Program Files / ProgramData folder, and, when you typed the path yourself, **inside `IIS_ALLOWED_PATH_PREFIXES`**.
3. `IIS_BACKUP_ROOT` is outside `IIS_SITE_PATH`.
4. IIS cross-check: when you configured a path and/or pool, they must equal what IIS reports for the site. A mismatch stops the pipeline (wrong-folder / wrong-pool protection).
5. The publish output is not empty and contains `web.config` (unless disabled).
6. An existing deployment is only overwritten after a backup was taken moments earlier in the same run.

## Backup
`<IIS_BACKUP_ROOT>\<APP_NAME>\yyyy-MM-dd_HHmmss\` with a `backup-manifest.txt` (build, commit, environment) and a pointer `last-backup.txt`. Only the newest `IIS_BACKUP_KEEP` timestamp folders are kept; nothing else in the backup root is ever deleted. Preserved paths are not part of the backup (they are never replaced).

## Deploy strategies
| `IIS_STRATEGY` | What happens |
|---|---|
| `app_offline` (default) | `app_offline.htm` is placed (ASP.NET Core unloads, visitors get a friendly 503), files are mirrored, the file is removed, the pool is recycled. The pool keeps running, so locked files are the only risk. |
| `stop_pool` | **Only the configured pool** is stopped (IIS stays up), files are copied, the pool is started in the Recycle stage. |

`IIS_PURGE: true` mirrors (removes files that are not in the new build) while **never** touching `IIS_PRESERVE_PATHS` (logs, uploads, `appsettings.Production.json`, ... add `web.config` here to keep the server's copy). `false` copies over only. robocopy retries locked files; exit codes 8+ fail the stage.
`IIS_GRANT_PERMISSIONS: true` grants the pool identity (`IIS AppPool\<pool>`) read access and Modify on `IIS_WRITABLE_PATHS`.

## Rollback
Automatic after a failure in Deploy / Recycle / Health Check (`ROLLBACK_ENABLED`): stop pool or app_offline, restore the backup (mirror, preserved paths untouched), recycle the pool, health check again. Each step is logged. `ACTION=rollback` does the same on demand. Manual run on the server (PowerShell as Administrator, environment variables from `config/iis.env.example` loaded): `.\scripts\iis-rollback.ps1` or `.\scripts\iis-rollback.ps1 -BackupPath D:\Backups\IIS\my-app\2026-10-03_083000`.
If the failed deployment was the first one there is no backup; the pipeline reports that clearly.

## Troubleshooting
| Symptom | Cause / fix |
|---|---|
| `not a local Administrator` | run the agent service under an administrator account |
| `IIS site ... points to ... but IIS_SITE_PATH is ...` | fix `IIS_SITE_PATH` (or the site's physical path) |
| `outside IIS_ALLOWED_PATH_PREFIXES` | add your hosting root, e.g. `D:\\Sites` |
| `robocopy failed (exit code 8+)` | locked files or no permission: Modify rights for the agent account, antivirus exclusions, keep `app_offline` |
| `web.config is missing` | `DOTNET_PROJECT` is not a web project |
| HTTP 500.30 / 502.5 after deploy | Hosting Bundle / runtime missing, or a wrong preserved config file; the pipeline rolls back |
| `Get-Website` not found | install IIS Management Scripts and Tools |
