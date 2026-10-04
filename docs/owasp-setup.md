# OWASP Dependency-Check setup

Runs on the Linux build agent (`scripts/owasp-scan.sh`) and writes `reports/owasp/dependency-check-report.html|json|xml`, archived by Jenkins.

## Install (once per agent)
1. Download the CLI release zip from the Dependency-Check GitHub releases page and unzip, e.g. to `/opt/dependency-check`.
2. Either put `/opt/dependency-check/bin` on the Jenkins user's PATH or set `OWASP_DC_HOME: '/opt/dependency-check'`.
3. Install `jq` (`apt install jq`) and a Java runtime.
4. Request a free **NVD API key** (nvd.nist.gov) and store it as a Jenkins Secret text credential -> `OWASP_NVD_API_KEY_CRED_ID`. Without a key the first database download is very slow and rate-limited.
5. Set `OWASP_DATA_DIR` to a persistent folder (e.g. `/var/lib/dependency-check-data`) so the NVD data is cached between builds.

If the CLI is not found the stage **fails with setup instructions**. It is never skipped silently; use `ENABLE_OWASP: false` if the application should not be scanned.

## Policy
The CVSS gate is evaluated from the JSON report:

* `OWASP_FAIL_THRESHOLD: 7` -> any vulnerability with CVSS >= 7 = FAIL (use `9` for critical only, `11` for report-only).
* A FAIL goes through `OWASP_POLICY` (`fail` | `unstable` | `ignore`) in the Security Gate.

## Options
`OWASP_SCAN_PATH` (default `.`), `OWASP_EXTRA_ARGS` (e.g. `--exclude **/node_modules/** --suppression config/suppressions.xml`).

## Troubleshooting
| Symptom | Fix |
|---|---|
| exit code 10, NVD download error | check network/proxy, add the API key, retry (first run can take a long time) |
| every build is slow | `OWASP_DATA_DIR` is missing or not persistent |
| false positives | add a suppression file via `OWASP_EXTRA_ARGS` and review it in code review |
