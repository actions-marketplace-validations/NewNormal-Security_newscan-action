# Security policy

## Reporting a vulnerability

Email **support@newnormalsecurity.com** with "SECURITY" in the subject, or use GitHub's
[private vulnerability reporting](https://github.com/NewNormal-Security/newscan-action/security/advisories/new).
Please don't open a public issue.

Include what you'd want to receive: affected version, steps to reproduce, and impact. We aim to
acknowledge within 2 business days and to ship a fix or a mitigation before any public disclosure.
We'll credit you unless you'd rather stay anonymous.

This policy covers this action and the NewScan scanner itself. NewScan releases daily
(`YY.MM.DD` versions) — please confirm against the current release before reporting.

## Scope note

NewScan is a scanner: only run it against systems you own or are explicitly authorised to test.

Scan results stay on your machine — the scanner does not upload findings or reports (the hosted
Teams layer is opt-in and separate). It does make two outbound calls to us: a daily version check,
which is a plain read you can disable with `NEWSCAN_UPDATE_CHECK=0`, and — in this action only — the
license-for-pull-token exchange that fetches the image.
