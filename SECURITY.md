# Security policy

TINY ART is a test-mode service operated by Canary Tech Labs S.L. for Asociación Cultural Nexus Gaia.

## Report a vulnerability

Send reports privately to **security@tinyart.es**. Please do not open a public issue for a security problem. Include the endpoint, the steps to reproduce and the time of the test. Do not include real personal data.

For misuse of the service or content that breaks the rules, write to `abuse@tinyart.es`. For general questions, `support@tinyart.es`.

[PLACEHOLDER: add a GitHub private vulnerability reporting link once the repository exists and the feature is enabled.]

## security.txt (RFC 9116 style)

A machine-readable version is planned at `https://tinyart.es/.well-known/security.txt` (not published yet). Intended content:

```
Contact: mailto:security@tinyart.es
Expires: [PLACEHOLDER: date less than one year ahead, ISO 8601]
Preferred-Languages: en, es
Canonical: https://tinyart.es/.well-known/security.txt
```

Format reference: https://www.rfc-editor.org/rfc/rfc9116

## What to expect

[PLACEHOLDER: response times to be set by the owner. No time frame is promised here.]

## Scope

In scope: the endpoints listed in `README.md` on `mcp.tinyart.es` and `tinyart.es`.
Out of scope: denial of service, social engineering, tests against other people's accounts, and anything outside those hosts.

## Good-faith testing

Use only your own test-mode identity. Do not try to read another identity's bid. Do not publish a vulnerability before it has been fixed.

## Secrets

Never post an API key or an owner secret in an issue. If one was shown to anyone by mistake, register a new identity and tell us the old one should be revoked.
