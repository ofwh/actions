# AI agent instructions (repo-specific)

## Repo purpose (how it’s used)
- This repo hosts **GitHub Actions automation** that runs on schedules and/or API-triggered workflows to implement recurring tasks.
- Typical example: scheduled check for remote file updates; if changed, download the new file and send a Telegram notification.

## Documentation-first (required here)
- Before implementing or changing any automation, **write/update a Markdown doc under `docs/`** describing:
  - Requirements / expected behavior
  - Implementation steps & data flow
  - Related files (workflows, scripts, `configs/`) and how to configure them
- Use `docs/automation-playbook.md` as the baseline structure.

## Requirement workflow (step-by-step)
1. **Create requirement folder**: Organize each requirement under `docs/<requirement-name>/`
2. **Provide original requirements**: Place user's original requirement document as `docs/<requirement-name>/<original-doc>.md`
3. **Generate specification**: Based on the original document, create `specification.md` in the same folder covering:
   - Requirement details & acceptance criteria
   - Implementation approach & data flow
   - Configuration & file changes needed
4. **Optional TODO tracking**: If follow-up tasks are planned, create `TODO.md` in the same folder (skip if not needed)
5. **Review & confirm**: **Wait for developer approval** of `specification.md` before starting implementation
6. **Implement**: Proceed with code generation only after specification is confirmed

## Current repo layout (scan ignores `.git`)
- `.github/workflows/`: workflow YAMLs (schedule, `workflow_dispatch`, API-triggered)
- `docs/`: automation docs (start with `docs/automation-playbook.md`)
- `configs/`: non-secret config files (YAML/JSON)
- `scripts/module/`: task-specific entrypoints/modules
- `scripts/shared/`: reusable building blocks
  - `scripts/shared/detectors/`: change/version detection logic
  - `scripts/shared/utils/`: generic utilities
- `.claude/`: Claude configuration; see `.claude/instructions.md` for the canonical instructions link
- `node_modules/`: local dependencies (ignored by git)

## Where things go
- Workflows: `.github/workflows/*.yml` (schedule, `workflow_dispatch`, and other triggers as needed)
- Scripts (preferred): `scripts/` using **Node.js LTS**
- Config/data (non-secret): `configs/` (YAML/JSON or other easy-to-parse formats)
- Docs: `docs/`

## Script conventions (preferred)
- Prefer **Node.js LTS** for automation scripts.
- Keep scripts modular and single-responsibility (task entrypoints in `scripts/module/`, reusable helpers in `scripts/shared/`).
- Avoid hard-coding environment-specific values; read from workflow inputs, repo variables, or `configs/` files.
- Prefer well-maintained **npm packages** over hand-rolled implementations (e.g., logger, HTTP client, YAML parsing).

## Configuration & reuse conventions
- Every automation should be **config-driven**:
  - Put non-secret settings in `configs/<task>.*` (commonly YAML/JSON) and document the schema in a `docs/` page.
  - Load config via a single place (entrypoint), then pass a typed/validated config object down to modules.
- Prefer **reusable building blocks** over copy/paste across tasks:
  - Put shared modules in `scripts/shared/` (especially `scripts/shared/detectors/` and `scripts/shared/utils/`).
  - Keep “pure logic” (diffing/version checks) separate from side effects (download, Telegram send, git commit).
  - Task entrypoints should mostly wire together shared modules + task-specific config.

## Secrets & security
- Never commit secrets (tokens, chat IDs, API keys).
- All sensitive values must be supplied via **GitHub Actions configuration** (Secrets / Variables) and passed as environment variables.
- `.gitignore` already ignores `.env` and `.env.*` (except `.env.example`); do not add real env files to git.

## Workflow authoring defaults
- Use explicit `permissions:` and pin action versions.
- If a workflow needs logic beyond a few lines, call a script in `scripts/module/` rather than embedding long inline shell.

## Implementation priority (how to build automations)
- Prefer **GitHub Marketplace Actions** first (reuse existing modules) when they fully satisfy the requirements.
  - Search Marketplace before writing custom code.
  - Example: sending Telegram notifications should use `appleboy/telegram-action` instead of a custom Telegram script.
- If no suitable Action exists, use **small, readable bash** for simple glue.
- If logic is complex or needs reuse/testing, implement as **Node.js LTS scripts** under `scripts/module/` and shared helpers under `scripts/shared/`.
