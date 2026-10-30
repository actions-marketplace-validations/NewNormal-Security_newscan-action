# Security policy

How to report a vulnerability in this action or in the NewScan scanner.

## Reporting

Email **support@newnormalsecurity.com** with "SECURITY" in the subject, or use GitHub's
[private vulnerability reporting](https://github.com/NewNormal-Security/newscan-action/security/advisories/new).
Do not open a public issue.

Include the affected version, steps to reproduce, and impact. We aim to acknowledge within
2 business days and ship a fix or mitigation before any public disclosure, and credit you unless you
prefer to stay anonymous. NewScan releases are date-versioned (`YY.MM.DD`) — confirm against the
current release before reporting.

## Scope

Only run NewScan against systems you own or are explicitly authorised to test.

Scan results stay on your machine; the scanner does not upload findings or reports. It makes two
outbound calls to us: a daily version check (a plain read, disable with `NEWSCAN_UPDATE_CHECK=0`)
and — in this action only — the license-for-pull-token exchange that fetches the image.
