# Security Policy

## Supported Versions

OpenCode Claude Bridge is pre-1.0 and tracks the `main` branch. Security fixes
land on `main` and are released as part of the next patch version.

| Version | Supported |
|---------|-----------|
| main    | ✅ yes    |
| < 0.2   | ❌ no     |

## Reporting a Vulnerability

Please report security vulnerabilities **privately**. Do not open a public
GitHub issue.

- Use GitHub's private vulnerability reporting (Security → Report a
  vulnerability) if available, or
- Email the maintainers through a GitHub security advisory.

Include:

- A description of the issue and its impact.
- Steps to reproduce (or a proof of concept).
- Affected version(s).

We aim to acknowledge reports within a few days and will coordinate a fix and
disclosure timeline with you.

## Secrets handling

OpenCode Claude Bridge is a local gateway. It must never ship real credentials:

- API keys / tokens are supplied at runtime via environment variables or
  `config.json`, which is intended to stay local and uncommitted.
- Examples in this repository use the placeholder `YOUR_OPENCODE_API_KEY`.
  Never substitute a real key in examples, tests, or docs.
- If you accidentally commit a secret, rotate it immediately and remove it from
  history before reporting.

## Scope notes

- The bridge binds to `127.0.0.1` by default. Exposing it on a public interface
  is unsupported unless you run it behind your own authenticated proxy.
- `POST /shutdown` is restricted to loopback clients. Keep it that way.
- Upstream calls leave your machine; transport is HTTPS to the configured
  OpenCode Zen base URL.