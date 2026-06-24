# claude-code-codex-skill

A Claude Code **skill** (not an app, dashboard, or service) that delegates tasks to the
**OpenAI Codex CLI** (GPT-5.5) as background agents — for precision coding, code review,
deliberation, and second opinions. This is a Canibuild-Ops fork of the upstream
`tomc98/claude-code-codex-skill` (`git remote -v` shows `upstream`); it powers the `/codex`
skill across the org.

Org-wide policy (Key Vault, SSO, push-to-main, stack defaults, Mason context) lives in
`~/.claude/CLAUDE.md` — not repeated here. This file is repo-specific only.

## What it does

- Wraps `codex exec` non-interactively, capturing the session id and clean final output.
- Two modes: **think** (read-only sandbox + web search, ephemeral) for analysis/review/second
  opinions; **run** (full-access sandbox + web search, persisted) for code changes. Plus
  `resume` (continue a session by id or `--last`) and `review` (Codex's built-in review).
- Codex is launched in the background; results are collected via `TaskOutput`.

## Structure (key files)

- `SKILL.md` — the skill prompt loaded by Claude Code. Frontmatter `name: codex`,
  `allowed-tools`, and the description. Edit this to change skill behavior/guidance.
- `scripts/codex.sh` — the only executable. Wrapper around `codex exec`. Dispatches on the
  first arg (`run` | `think` | `resume` | `review`). `parse_common_flags` + `build_cmd`
  assemble the CLI; mode functions set defaults (run → `danger-full-access`, think → `read-only`
  + `--ephemeral`, both → web search on). Output: `SESSION: <id>\n---\n<response>`.
- `references/cli-reference.md` — Codex CLI flag reference.
- `references/prompt-engineering.md` — prompt templates for complex tasks.
- `README.md` — user-facing install + usage docs.
- `AGENTS.md` — Codex's own org guidelines (mirrors org CLAUDE policy); synced from
  `canibuild-config`, do not hand-edit (commits show `chore: sync AGENTS.md from canibuild-config`).
- `.github/CODEOWNERS`, `.github/workflows/secret-scan.yml` — applied by the governance baseline.

## Invocation / install / test

- Invoked in Claude Code as `/codex <think|run|review|resume> ...` (see SKILL.md / README).
- Installed by cloning into `~/.claude/skills/codex` or via the `skill-manager` skill.
- Requires the Codex CLI on PATH (`npm install -g @openai/codex`) and `OPENAI_API_KEY` /
  `CODEX_API_KEY` in the environment. Codex itself is configured in `~/.codex/config.toml`
  (model, reasoning effort, sandbox), overridable per-call via `--model` / `--effort` / `--sandbox`.
- No build step or test suite. Smoke-test the wrapper directly, e.g.
  `scripts/codex.sh think "say hello" --dir .` and check for a `SESSION:` line.

## Conventions / gotchas

- **Self-healing by design:** SKILL.md authorizes editing any file under the skill base when
  Codex CLI behavior drifts. Keep `scripts/codex.sh` and the `references/*` flag docs in sync
  with the installed Codex CLI version.
- `codex.sh` uses `set -eo pipefail`; codex invocations are `|| true` so a non-zero Codex exit
  still emits captured output rather than aborting. stdin is redirected from `/dev/null` (codex
  hangs otherwise — see commit `1186e46`).
- Model/version strings live in README + references (e.g. GPT-5.5); bump them together when the
  CLI/model changes.
- This is open-source: no secrets, no Canibuild-internal details belong in committed files. The
  `secret-scan` workflow runs on push.
- `master`/`main` carries an `upstream` remote — push only to `origin` (Canibuild-Ops fork).
