# NewScan Security Scan — GitHub Action

Run [NewScan](https://newnormalsecurity.com) as a CI security gate: it scans a deployed URL, writes
SARIF, and fails the job on findings at or above a severity you choose. **This packaged integration
is a NewScan Pro feature.** (Self-hosting NewScan and calling `python -m newscan` yourself stays
free — see the NewScan docs.)

## Usage

```yaml
- uses: NewNormal-Security/newscan-action@v1
  with:
    target: https://staging.example.com
    license: ${{ secrets.NEWSCAN_LICENSE }}   # your Pro license — the only secret you need
    fail-on: high
- if: always()
  uses: github/codeql-action/upload-sarif@v3
  with: { sarif_file: newscan.sarif }
```

A full deploy-gating workflow is in [`examples/security-gate.yml`](examples/security-gate.yml).

> **SARIF → code scanning:** `upload-sarif` surfaces findings in the repo's Security tab. That needs
> code scanning enabled — free on **public** repos, or **GitHub Advanced Security** on private ones.
> The example keeps the SARIF as a downloadable **artifact** regardless, and marks the upload
> `continue-on-error` so a repo without code scanning still passes (the scan gate itself already
> fails the job on findings).

## Inputs

| Input | Default | Description |
|---|---|---|
| `target` | — (required) | URL to scan |
| `license` | — (required) | Pro license token; store as the `NEWSCAN_LICENSE` secret |
| `scan-mode` | `api` | `api` \| `web` \| `network` \| `wifi` \| `segmentation` |
| `profile` | `quick` | `quick` \| `baseline` (no key) · `full` \| `safe` (BYOK) |
| `fail-on` | `high` | Severity that fails the job: `critical`…`info` |
| `sarif` | `newscan.sarif` | Where to write SARIF 2.1.0 (in the workspace) |
| `extra-headers` | — | Extra request headers (one `Name: Value` per line) to reach a protected pre-prod target — e.g. `x-vercel-protection-bypass: <token>`. Sent on every request. |
| `goal`, `model`, `token-budget`, `config` | — | Optional; map to the `python -m newscan` flags |

## Outputs

| Output | Description |
|---|---|
| `sarif` | Path to the written SARIF file |
| `exit-code` | `0` pass · `1` finding at/above `fail-on` · `2` error |

## Scan pre-production, not production

The point of this action is to catch issues **before** a release reaches production — so point it at
a **release candidate the scanner can reach cleanly**, never at hardened prod (a production WAF/bot
protection will block the scan and give you misleading results). Two patterns:

**A — Spin the app up in the CI job (recommended default).** Start the API (and a throwaway DB) as a
job `service`, scan `http://localhost:PORT`. No external environment, no WAF, isolated, fast, and it
scans the exact commit. The action runs `--network host`, so `localhost` reaches the service. See
[`examples/ephemeral-service.yml`](examples/ephemeral-service.yml).

**B — Scan the per-PR preview deploy.** Vercel/Netlify/Cloudflare/etc. mint an ephemeral preview URL
per PR — that *is* your pre-prod environment (no standing "staging" box to maintain). Scan the
preview URL and make it a **required status check** so a PR can't merge (→ prod) unless it passes.
See [`examples/preview-gate.yml`](examples/preview-gate.yml).

Protected previews (login wall / WAF) need the scanner let in — use `extra-headers`:
- **Vercel:** enable **Deployment Protection → Protection Bypass for Automation**, store the token as
  a secret, and pass `extra-headers: "x-vercel-protection-bypass: ${{ secrets.VERCEL_AUTOMATION_BYPASS_SECRET }}"`.
- **Generally:** allowlist the scanner (its `User-Agent: NewScan/…`, its IP, or a shared secret
  header) in the firewall **for the preview only**, and keep aggressive challenge modes off there.

If the scanner is blocked, it now says so explicitly (an *"Scan is being blocked by the target"*
observation) instead of a misleading green "0 findings" — that's your cue to use one of the above.

## How it works

The action exchanges your Pro license for a short-lived, read-only token to pull the private NewScan
image, then runs the scan through a Pro-gated entrypoint. Your license is validated live, so a
revoked or expired license can't run the integration.

- `quick`/`api` needs no provider key and no browser — the fastest gate.
- `web` uses the image's bundled headless Chromium.
- `full`/`safe` use the AI layer: add your provider key as a secret and pass it via `env:`.

## Getting a license

Buy or manage NewScan Pro at <https://newnormalsecurity.com/online-packs#pricing>, then add the license token as
an Actions secret named `NEWSCAN_LICENSE` in your repository or organization.
