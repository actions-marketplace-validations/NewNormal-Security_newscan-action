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

## Inputs

| Input | Default | Description |
|---|---|---|
| `target` | — (required) | URL to scan |
| `license` | — (required) | Pro license token; store as the `NEWSCAN_LICENSE` secret |
| `scan-mode` | `api` | `api` \| `web` \| `network` \| `wifi` \| `segmentation` |
| `profile` | `quick` | `quick` \| `baseline` (no key) · `full` \| `safe` (BYOK) |
| `fail-on` | `high` | Severity that fails the job: `critical`…`info` |
| `sarif` | `newscan.sarif` | Where to write SARIF 2.1.0 (in the workspace) |
| `goal`, `model`, `token-budget`, `config` | — | Optional; map to the `python -m newscan` flags |

## Outputs

| Output | Description |
|---|---|
| `sarif` | Path to the written SARIF file |
| `exit-code` | `0` pass · `1` finding at/above `fail-on` · `2` error |

## How it works

The action exchanges your Pro license for a short-lived, read-only token to pull the private NewScan
image, then runs the scan through a Pro-gated entrypoint. Your license is validated live, so a
revoked or expired license can't run the integration.

- `quick`/`api` needs no provider key and no browser — the fastest gate.
- `web` uses the image's bundled headless Chromium.
- `full`/`safe` use the AI layer: add your provider key as a secret and pass it via `env:`.

## Getting a license

Buy or manage NewScan Pro at <https://newnormalsecurity.com/pricing>, then add the license token as
an Actions secret named `NEWSCAN_LICENSE` in your repository or organization.
