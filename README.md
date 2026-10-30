# NewScan Security Scan — GitHub Action

Run a [NewScan](https://newnormalsecurity.com/newscan?utm_campaign=github) scan on every PR or
deploy, get SARIF, fail the build on findings at or above a severity you choose. NewScan is the
self-hosted scanner for APIs, web apps, network, Wi-Fi and segmentation that reproduces every
finding before recording it; this action is the CI gate and requires **NewScan Pro**.

- **Get NewScan free:** <https://newnormalsecurity.com/newscan?utm_campaign=github#get>
- **Pro (this action + out-of-band detection):** <https://newnormalsecurity.com/online-packs?utm_campaign=github#pricing>
- **Roadmap:** <https://newnormalsecurity.com/roadmap?utm_campaign=github>

## Usage

```yaml
- uses: NewNormal-Security/newscan-action@v1
  with:
    target: https://staging.example.com
    license: ${{ secrets.NEWSCAN_LICENSE }}   # the only secret you need
    fail-on: high
- if: always()
  uses: github/codeql-action/upload-sarif@v4
  with: { sarif_file: newscan.sarif }
```

Complete workflows in [`examples/`](examples/): [deploy gate](examples/security-gate.yml),
[ephemeral service](examples/ephemeral-service.yml), [per-PR preview gate](examples/preview-gate.yml).

`upload-sarif` needs code scanning enabled (free on public repos, GitHub Advanced Security on private
ones). The examples keep the SARIF as an artifact regardless and mark the upload `continue-on-error`;
the gate itself already fails the job on findings.

## Inputs

| Input | Default | Description |
|---|---|---|
| `target` | required | http(s) URL to scan |
| `license` | required | Pro license token; store as the `NEWSCAN_LICENSE` secret |
| `scan-mode` | `api` | `api` \| `web` \| `network` \| `wifi` \| `segmentation` |
| `profile` | `quick` | `quick` \| `baseline` (no provider key) · `full` \| `safe` (bring your own key via `env:`) |
| `fail-on` | `high` | Severity that fails the job: `critical` \| `high` \| `medium` \| `low` \| `info` |
| `sarif` | `newscan.sarif` | Path (within the workspace) to write SARIF 2.1.0 |
| `extra-headers` | — | One `Name: Value` per line, sent on every request (preview protection / WAF allow rule) |
| `goal`, `model`, `token-budget`, `config` | — | Optional; passed through as the matching `python -m newscan` flags. `config` is a UI-exported JSON config (auth, scope, roles) |
| `api-base` | `https://newnormalsecurity.com` | License/pull-token service base URL (testing only) |

## Outputs

| Output | Description |
|---|---|
| `sarif` | Path to the written SARIF file |
| `exit-code` | `0` pass · `1` finding at/above `fail-on` · `2` error |

## Licensing

1. Get Pro at the [plan page](https://newnormalsecurity.com/online-packs?utm_campaign=github#pricing),
   or reuse any Pro license you already have (even one activated on your workstation).
2. Repository or organization **Settings → Secrets and variables → Actions** → secret `NEWSCAN_LICENSE`.
3. Pass it only to the `license` input. Never echo it or add it to application configuration.

Licenses carry 3 seats (one per device or repository). The action sets `NEWSCAN_LICENSE_KEY` and a
stable runner identity (`NEWSCAN_DEVICE_ID=gha-<owner>/<repo>`) inside the scanner container; a
token bound to another device claims a seat for the repository automatically on the first run. Out
of seats → the gate fails with a pointer to release one from the
[account page](https://newnormalsecurity.com/account). Each run exchanges the license for a
short-lived read-only image-pull credential (`POST /api/ci/pull-token`), so an expired or revoked
token stops the gate before the scan starts.

### Calling the container directly (no action)

```bash
docker run --rm \
  -e NEWSCAN_LICENSE_KEY="$NEWSCAN_LICENSE" \
  -v "$PWD:/work" -w /work \
  ghcr.io/newnormal-security/newscan@sha256:<digest> \
  python -m newscan.ci_gate https://staging.example.com \
    --scan-mode api --profile quick --fail-on high --sarif /work/newscan.sarif
```

On GitHub the runner identity derives from `GITHUB_REPOSITORY` automatically; elsewhere set
`NEWSCAN_DEVICE_ID` to a fixed string unique to the pipeline (Jenkins: `jenkins-<controller>-<job>`).
Pull credential: `POST /api/ci/pull-token` with `{"license": "<token>"}`, or your org's own
read-only PAT.

## Scan pre-production, not production

Point the gate at a release candidate the scanner can reach cleanly — never hardened prod, where a
WAF/bot protection blocks the scan and gives misleading results. Two patterns:

- **Spin the app up in the job** (recommended): start the API (and a throwaway DB) as a job
  `service`, scan `http://localhost:PORT`. The action runs with `--network host`, so `localhost`
  reaches the service. See [`examples/ephemeral-service.yml`](examples/ephemeral-service.yml).
- **Scan the per-PR preview deploy** (Vercel/Netlify/Cloudflare) and make it a required status
  check. See [`examples/preview-gate.yml`](examples/preview-gate.yml). Protected previews need the
  scanner let in via `extra-headers` — Vercel: enable *Deployment Protection → Protection Bypass for
  Automation* and pass `extra-headers: "x-vercel-protection-bypass: ${{ secrets.VERCEL_AUTOMATION_BYPASS_SECRET }}"`;
  generally, allowlist the scanner (`User-Agent: NewScan/…`, IP, or a shared header) for the preview only.

A blocked scan reports a *"Scan is being blocked by the target"* observation rather than a misleading
green "0 findings".

**Only scan what you own or are authorised to test.**

## Free vs Pro

Local, in-band scanning is free: the whole deterministic engine, any number of machines, including
`python -m newscan` in your own CI. Pro covers what needs a listener on the public internet
(out-of-band detection) plus packaged integrations like this action.

## Reporting a vulnerability

Don't open a public issue — see [docs/SECURITY.md](docs/SECURITY.md).

## License

This action wrapper is [MIT](LICENSE). NewScan itself is proprietary, free to download and run
locally under the [EULA](https://newnormalsecurity.com/eula?utm_campaign=github).

---
Built by [NewNormal Security](https://newnormalsecurity.com/?utm_campaign=github).
