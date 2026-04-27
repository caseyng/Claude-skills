# Changelog

## 2.0.0 — 2026-04-27

### Changed
- **Invocation API redesigned** (breaking): removed implicit mode detection based on conversation context
  - `/skill:session-handoff` (no arg) → always generates a handoff
  - `/skill:session-handoff [file]` → always resumes from that file path
- Resume mode now reads the handoff file via Read tool before processing; no longer requires the user to paste the content into chat

### Added
- **Context pressure auto-alert**: when the system compression message appears, Claude emits a warning and prompts the user to run `/skill:session-handoff` before context is lost — fires before any other response, without requiring invocation
- README.md with invocation patterns, mode descriptions, and design rationale
- CHANGELOG.md (this file)
- Version field in SKILL.md frontmatter

## 1.0.0 — initial

- Two implicit modes: generating and receiving a handoff
- Mode 0 live-capture guidance during session
- Handoff structure template with cut test and inclusion test
- Compression rules for Claude-optimised handoff format
