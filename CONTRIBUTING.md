# Contributing to OpenCode Claude Bridge

Thanks for your interest in improving OpenCode Claude Bridge! This document covers
how to get a change merged.

## Getting started

1. Fork and clone the repository.
2. No build step or dependency install is required — the bridge runs on Node.js 18+
   with zero npm dependencies.
3. Run the checks locally before opening a pull request:

   ```bash
   npm run check   # syntax check: node --check server.js
   npm test        # node --test
   ```

## Project identity

- This project is **OpenCode Claude Bridge** (package name `opencode-claude-bridge`).
  It is an independent, community-maintained compatibility bridge. It is **not** an
  official OpenCode project and does not claim to be one.
- OpenCode Zen is a supported upstream / platform dependency. Mention it where it
  is technically relevant (configuration, routing, upstream errors), but do not
  imply endorsement or ownership.
- Do **not** rename the project or change its public directory layout without
  discussion.

## What we accept

- Bug fixes with a root-cause explanation.
- Protocol-translation correctness fixes (Anthropic ↔ OpenAI/OpenCode).
- Model-rotation and fallback improvements that preserve existing behavior
  unless a change is required for correctness, security, or maintainability.
- Documentation, examples, and tests.

## What we do not accept

- Fabricated features that are not implemented.
- Changes that overwrite working core behavior without a stated reason.
- Commits that contain real API keys, tokens, or other secrets. Use the
  `YOUR_OPENCODE_API_KEY` placeholder in examples.

## Pull request guidelines

- Keep diffs focused. One logical change per PR.
- Add or update tests for behavior changes. Rotation and effort-mapping logic
  have unit + integration coverage in `test/server.test.js`.
- Update `README.md` and `config.example.json` when you change configuration keys.
- Update `CHANGELOG.md` under `## [Unreleased]` for user-visible changes.

## Reporting security issues

Do **not** open a public issue for vulnerabilities. See
[SECURITY.md](SECURITY.md) for private disclosure instructions.

## Code style

- Plain Node.js, no transpilation. Match the surrounding style (2-space indent,
  double quotes, `camelCase`).
- Prefer the smallest change that works. Avoid new dependencies.