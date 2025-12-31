# Automation playbook (write docs first)

This repo is used to build GitHub Actions-based automation (scheduled runs and API-triggered runs) that executes scripts (preferably Node.js LTS) and integrates with external services (e.g., Telegram).

## Before you start: Requirement workflow
Before filling in this playbook, follow the requirement workflow documented in `.github/copilot-instructions.md`:
1. Create a requirement folder: `docs/<requirement-name>/`
2. Place original requirement document: `docs/<requirement-name>/<original-doc>.md`
3. Generate `specification.md` with requirement details, implementation approach, and configuration needs
4. Create `TODO.md` if there are follow-up tasks (optional)
5. **Get developer approval** of the specification before proceeding

Once the specification is confirmed, use this playbook template to document the implementation details below.

## 1) Requirement checklist (fill this in first)
- **Goal**: what problem this automation solves.
- **Triggers**:
  - Schedule: cron expression(s)
  - Manual: `workflow_dispatch` inputs
  - API-triggered: which event/endpoint (e.g., repository dispatch) and payload shape
- **Inputs** (non-secret): repo variables / workflow inputs / config file keys.
- **Secrets** (secret): which GitHub Secrets are required (names only).
- **Outputs**: artifacts, commits/PRs, notifications (Telegram), logs.
- **Failure modes**: what should happen on network errors / rate limits / malformed data.

## 2) Implementation plan (data flow)
Describe the end-to-end flow in a few bullets. Example pattern:
1. Fetch a “remote version indicator” (ETag / Last-Modified / checksum / version file).
2. Compare against previously stored state (artifact / cache / repo file, depending on the chosen approach).
3. If changed: download the new file.
4. Publish result (artifact or commit) and notify Telegram.

## 2.1) Implementation approach (priority order)
Document which approach you chose and why:
1. Prefer GitHub Marketplace Actions (existing modules) when they fully satisfy the requirements.
   - Example: Telegram notifications should use `appleboy/telegram-action`.
2. Otherwise use minimal bash for simple glue steps.
3. For complex logic/reuse, implement Node.js LTS scripts in `scripts/module/` and shared helpers in `scripts/shared/`.

## 3) Repository layout (where files live)
- Workflows: `.github/workflows/<name>.yml`
- Script entrypoints: `scripts/module/<task>/index.js` (or similar)
- Supporting modules: `scripts/module/<task>/*.js`
- Shared reusable modules: `scripts/shared/**` (recommended)
- Non-secret configuration:
  - `configs/` for settings (URLs, paths, feature flags) in YAML/JSON (or similar)

## 4) Configuration & secrets
- Put **non-secret** settings in `configs/` and document their schema here.
- Put **secrets** in GitHub Actions Secrets/Variables. Document required names here (but never their values), e.g.:
  - `TELEGRAM_BOT_TOKEN`
  - `TELEGRAM_CHAT_ID`

## 4.1) Modularity & reuse (required)
- Keep the task entrypoint thin: load/validate config once, then call modules.
- Prefer **configuration-driven** behavior (URLs, filenames, thresholds, chat IDs via secrets, etc.), not hard-coded constants.
- Avoid duplication across automations by extracting shared pieces into `scripts/shared/`, for example:
  - `telegram` client (sendMessage, format)
  - HTTP fetch + retries/timeouts
  - change detection (ETag/Last-Modified/hash)
  - state persistence strategy (artifact/cache/repo file)

## 5) Example: “check remote file updates → download → Telegram notify”
Document for this automation should include:
- Remote URL(s) and what defines “changed” (ETag/Last-Modified/hash)
- Where state is stored between runs (and why)
- Which script module does:
  - change detection
  - download
  - message formatting
  - telegram send
- Which workflow passes env/inputs/secrets to the script

## 6) Files to reference in this doc (fill in once implemented)
- Workflow: `.github/workflows/<name>.yml`
- Script entry: `scripts/module/<task>/index.js`
- Config: `configs/<task>.yaml` / `configs/<task>.json` (or similar)
