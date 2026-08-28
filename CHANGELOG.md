# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.1] - 2026-08-29

### Changed
- Multi-model routing: DeepSeek V4 plus additional OpenCode Zen models
  (MoMo, Hy3, Nemotron, Laguna, and others) are supported via the `models`
  / `fallbackModels` lists.
- Windows autostart task and tray icon renamed to `OpenCodeClaudeBridge`.

### Added
- `config.example.json` — annotated, secret-free configuration template.
- `CONTRIBUTING.md`, `SECURITY.md`, and this `CHANGELOG.md`.
- Explicit model-rotation policy: rotate on `429 / 500 / 502 / 503 / 504` and
  on detected `400` "model unavailable" / provider-routing failures; do **not**
  rotate on `401 / 403` or other `400` errors.
- Unit and integration tests for the rotation policy in `test/server.test.js`.

### Fixed
- Effort mapping now emits DeepSeek `reasoning_effort: "high"` for non-low
  requests (previously asserted `"max"` in tests).

## [0.2.0]

- Initial public-ready bridge: Anthropic `/v1/messages` ↔ OpenCode Zen
  OpenAI-compatible `/v1/chat/completions` translation, SSE streaming, reasoning
  cache replay, effort mapping, and cross-platform autostart.

[Unreleased]: https://github.com/your-org/opencode-claude-bridge/compare/v0.2.1...HEAD
[0.2.1]: https://github.com/your-org/opencode-claude-bridge/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/your-org/opencode-claude-bridge/releases/tag/v0.2.0