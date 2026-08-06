# NewScan Security Scan — GitHub Action

**All signal, no noise.** [NewScan](https://newnormalsecurity.com/newscan?utm_campaign=github) is a free, self-hosted
scanner for penetration tests — **APIs, web apps, network & infrastructure, Wi-Fi, and segmentation
in one tool**. It runs the full deterministic scan on your box, reproduces every finding before it
records it, and only then offers an optional AI pass (bring your own key). Findings carry one
calibrated severity that maps to PCI DSS / SOC 2 / ISO 27001 / HITRUST / HIPAA / NIST 800-53 ratings and
remediation SLAs.

**This repo is the CI gate**: run a scan on every PR or deploy, get SARIF, fail the build on
findings at or above a severity you choose.

- 🡒 **Get NewScan free:** <https://newnormalsecurity.com/newscan?utm_campaign=github#get> — email to
  download, no account needed to run it locally.
- 🡒 **Pro (this action, plus out-of-band detection):**
  <https://newnormalsecurity.com/online-packs?utm_campaign=github#pricing>
- 🡒 **Roadmap:** <https://newnormalsecurity.com/roadmap?utm_campaign=github>

## Usage

```yaml
- uses: NewNormal-Security/newscan-action@v1
  with:
    target: https://staging.example.com
    license: ${{ secrets.NEWSCAN_LICENSE }}   # your Pro license — the only secret you need
    fail-on: high
- if: always()
  uses: github/codeql-action/upload-sarif@v4
  with: { sarif_file: newscan.sarif }
```

Three complete workflows are in [`examples/`](examples/): a
[deploy gate](examples/security-gate.yml), an [ephemeral service](examples/ephemeral-service.yml),
and a [per-PR preview gate](examples/preview-gate.yml).

> **SARIF → code scanning:** `upload-sarif` surfaces findings in the repo's Security tab. That needs
> code scanning enabled — free on **public** repos, or **GitHub Advanced Security** on private ones.
> The examples keep the SARIF as a downloadable **artifact** regardless, and mark the upload
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

If the scanner is blocked it says so explicitly (a *"Scan is being blocked by the target"*
observation) instead of a misleading green "0 findings" — that's your cue to use one of the above.

**Only scan what you own or are authorised to test.** The gate is built for your own pre-prod
environments.

## How it works

The action exchanges your Pro license for a short-lived, read-only token to pull the NewScan image,
then runs the scan through a Pro-gated entrypoint. Your license is validated live, so a revoked or
expired license can't run the integration.

- `quick`/`api` needs no provider key and no browser — the fastest gate.
- `web` uses the image's bundled headless Chromium.
- `full`/`safe` use the AI layer: add your provider key as a secret and pass it via `env:`.

## Free vs Pro

**Local, in-band scanning is free** — the whole deterministic engine, on as many machines as you
like, no license. **Pro** covers what needs a listener on the public internet (out-of-band /
blind-vulnerability detection) plus packaged integrations like this action. Self-hosting NewScan and
calling `python -m newscan` in your own CI stays free.

Get a license at <https://newnormalsecurity.com/online-packs?utm_campaign=github#pricing>, then add the token as an
Actions secret named `NEWSCAN_LICENSE` on the repository or organization.

## Reporting a vulnerability

Please don't open a public issue — see [SECURITY.md](SECURITY.md).

## License

This action wrapper is [MIT](LICENSE). NewScan itself is proprietary software, free to download and
run locally under the [EULA](https://newnormalsecurity.com/eula?utm_campaign=github).

---
Built by [NewNormal Security](https://newnormalsecurity.com/?utm_campaign=github).
