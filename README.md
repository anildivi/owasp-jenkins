# OWASP Dependency-Check — Jenkins Pipeline Demo

A minimal example showing two ways to integrate **OWASP Dependency-Check**
into a Jenkins pipeline.

## Project layout

```
owasp-jenkins-demo/
├── package.json          # sample Node app deps (express, lodash)
├── src/index.js          # tiny Express app
├── Jenkinsfile           # Option A: uses the Jenkins plugin + Global Tool
└── Jenkinsfile.docker    # Option B: uses the official Docker image directly
```

## Option A — `Jenkinsfile` (plugin-based)

**Setup required (one-time, in Jenkins):**

1. Install the **OWASP Dependency-Check** plugin
   (Manage Jenkins → Plugins → search "Dependency-Check").
2. Manage Jenkins → Tools → add a **Dependency-Check installation**
   named `Dependency-Check` (auto-install from GitHub is fine).
3. Point a Jenkins job (Pipeline / Multibranch Pipeline) at this repo —
   it will pick up the `Jenkinsfile` automatically.

**What it does:**
- Installs deps, runs unit tests.
- Runs Dependency-Check, fails the scan if CVSS ≥ 7.
- Publishes results into the Jenkins UI via `dependencyCheckPublisher`
  (you get a trend graph across builds, plus a findings table).
- Explicit "Quality Gate" stage stops the pipeline before any
  build/deploy stage runs if the scan failed.
- `post { failure {...} }` is where you'd wire up Slack/email alerts.

This is the better choice long-term: native UI integration, trend
graphs, and configurable thresholds (Unstable/Failed) in plugin settings.

## Option B — `Jenkinsfile.docker` (no plugin needed)

**Setup required:** just Docker available on the Jenkins agent. No
plugin, no Global Tool config.

**What it does:**
- Runs `npm ci` inside a `node:20-alpine` container.
- Runs the `owasp/dependency-check` Docker image directly against the
  workspace, mounting a `dc-data` folder so the CVE database is cached
  between builds (avoids re-downloading ~1GB every run).
- Publishes the HTML report via the `publishHTML` plugin step (requires
  the **HTML Publisher** plugin — commonly already installed).
- Fails the build the same way, via `--failOnCVSS 7`.

Use this if you want something working immediately without touching
Global Tool Configuration, or if your Jenkins agents are containerized
and ephemeral.

## Tuning the security gate

`--failOnCVSS 7` fails on High/Critical (CVSS 7.0+). Adjust to your
risk appetite:
- `--failOnCVSS 4` — fail on Medium and above (stricter)
- `--failOnCVSS 9` — only fail on Critical (looser)

You can also suppress known false positives with a
`dependency-check-suppression.xml` file and `--suppression <file>`.

## Adding OWASP ZAP (DAST) as a second stage

If you also want to scan the *running* app (not just dependencies),
add a stage that starts the app and runs ZAP's baseline scan against it:

```groovy
stage('OWASP ZAP Baseline Scan') {
    steps {
        sh '''
            npm start &
            sleep 5
            docker run --rm -v $(pwd):/zap/wrk/:rw \
                -t ghcr.io/zaproxy/zaproxy:stable \
                zap-baseline.py -t http://host.docker.internal:3000 \
                -r zap-report.html
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
        }
    }
}
```

Let me know if you want this merged into one full pipeline (SCA + DAST),
a Jenkins shared-library version for reuse across many repos, or
Slack/email notification wiring added.
