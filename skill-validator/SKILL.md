---
name: skill-validator
description: >-
  Install and run skill-validator against Cursor Agent Skills: structural
  check (--strict) and LLM score evaluate (prefer claude-cli). Covers Windows
  PATH for ~/.local/bin/claude, auth via claude login, and API-key fallbacks.
  Use when validating or scoring a SKILL.md, running skill-validator check or
  score evaluate, or debugging Claude Code CLI auth for scoring.
---

# skill-validator workflow

Validate a Cursor skill directory (must contain `SKILL.md`). Prefer the **directory** path, not only the file.

## Prerequisites

1. **skill-validator** (Go):
   ```powershell
   go install github.com/agent-ecosystem/skill-validator/cmd/skill-validator@latest
   ```
   Binary is typically at `~/go/bin/skill-validator` — ensure that is on `PATH`.

2. **For `score evaluate` only** — one LLM provider:
   | Provider | Flag | Auth |
   |----------|------|------|
   | Claude Code CLI (preferred on this machine) | `--provider claude-cli` | `claude login` (no API key) |
   | Anthropic API | default | `ANTHROPIC_API_KEY` |
   | OpenAI API | `--provider openai` | `OPENAI_API_KEY` |

## Windows: Claude Code CLI

- Installer: `irm https://claude.ai/install.ps1 | iex` (or `winget install Anthropic.ClaudeCode`).
- CLI binary: `~/.local/bin/claude.exe` (Claude Code). **Not** the desktop app (`Anthropic.Claude` / `AnthropicClaude`).
- If `claude` is missing in the current shell:
  ```powershell
  $env:PATH = "$env:USERPROFILE\.local\bin;$env:PATH"
  ```
- Confirm auth before scoring:
  ```powershell
  claude auth status
  # need: "loggedIn": true
  # else: claude login
  ```

## Run checks (always)

```powershell
skill-validator check --strict -o markdown "PATH/TO/skill-dir/"
```

Skip `--emit-annotations` locally (GitHub Actions only). Report pass/fail and notable metrics (tokens, contamination).

## Run LLM scoring

```powershell
skill-validator score evaluate "PATH/TO/skill-dir/" --provider claude-cli -o markdown
```

Useful flags: `--skill-only`, `--full-content`, `--rescore`, `--model <name>`.

Cached results: `skill-validator score report "PATH/TO/skill-dir/" -o markdown`.

If `claude-cli` fails with empty/exit 1 → re-check `claude auth status` and PATH.

## Agent procedure

1. Resolve target to a skill **directory** containing `SKILL.md`.
2. Ensure `skill-validator` is on PATH; install with `go install` if missing.
3. Run `check --strict -o markdown`; summarize results to the user.
4. If the user wants scores (or asked for score evaluate):
   - Prefer `--provider claude-cli`: fix PATH, verify `loggedIn`, then evaluate.
   - Else use `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` if set.
   - If no provider available, stop and tell the user how to log in or set a key.
5. Summarize dimension scores and overall; do not invent scores without running the tool.

## Repo CI note (customer-docs)

PR workflow `validate-skills.yml` runs **check only** via `.github/scripts/validate-skills.sh` on changed `.cursor/skills/` dirs. Local `score evaluate` is optional and not part of that gate.
