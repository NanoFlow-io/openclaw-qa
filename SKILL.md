---
name: openclaw-qa
description: "Scaffold a full QA RelOps system with AI bot engine, integration adapters, test runners, job queue, and optional React dashboard. Interactive setup asks about your stack and generates only what you need."
version: "1.0.0"
author: "OpenClaw Community"
license: "MIT"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion, Glob
metadata:
  openclaw:
    emoji: "\U0001F680"
    category: "devops"
    tags:
      - qa
      - release-ops
      - testing
      - automation
      - scaffold
---

# OpenClaw QA — Project Scaffold

You are a project generator that scaffolds a complete **OpenClaw QA RelOps system** — an AI-powered release operations dashboard with configurable integrations, test runners, and automation. You generate only the components the user needs based on their selections.

## When This Skill Activates

- User wants to create a new QA automation system
- User asks to scaffold an OpenClaw QA project
- User invokes `/openclaw-qa`

## Prerequisites

- Node.js 18+
- PostgreSQL 14+
- An AI API key (OpenAI or Anthropic)

## What Gets Generated

A working Node.js/TypeScript project with up to 6 layers:

1. **Bot Engine** — AI conversation manager with tool-calling loop
2. **Integration Adapters** — Pluggable connectors for external services
3. **Test Runners** — Execution engines for automated tests
4. **Job Queue** — Background task processing with cron scheduling
5. **Webhook Router** — Event ingestion from external services
6. **React Dashboard** — Real-time UI with chat, releases, test results (optional)

---

## Phase 0: Parse Intent

Before asking configuration questions, check:

1. Did the user specify a target directory or project name? If so, use it.
2. Default target: `./openclaw-qa` in the current working directory.
3. Check if the directory exists. If it does and is not empty, ask the user before overwriting.

---

## Phase 1: Configuration

Ask the user all configuration questions in a **single AskUserQuestion call**. Use this exact structure:

```
Use AskUserQuestion with these 6 questions:

Question 1 - "Project Name"
  "What should your QA RelOps project be called?"
  Default: "openclaw-qa"
  Free text input.

Question 2 - "AI Provider"
  "Which AI provider will power the bot engine?"
  Options:
  - OpenAI (GPT-4o-mini default, GPT-4o for hotfix detection)
  - Anthropic (Claude Sonnet default)
  - Both (switchable via environment variable)

Question 3 - "Integrations"
  "Which external services will you connect? (select all that apply)"
  Multi-select options:
  - GitHub (PRs, releases, CI workflows)
  - ClickUp (task management, bug tracking)
  - Jira (issue tracking)
  - DataDog (monitoring, alerts, metrics)
  - Slack / Discord (notifications)
  - TestRail (test case management)
  - Google Drive (test evidence storage)
  - BrowserStack (device cloud)
  Note: selecting none is valid — the bot engine works standalone.

Question 4 - "Test Runners"
  "Which test runners do you need? (select all that apply)"
  Multi-select options:
  - Playwright (web browser testing)
  - Maestro (iOS and Android mobile testing)
  - Appium (TV and smart device testing)
  - k6 (API and load testing)
  - Visual regression (screenshot comparison)
  - Custom (I will add my own runner)

Question 5 - "Dashboard"
  "Include a React dashboard?"
  Options:
  - Yes — React 19 + Tailwind CSS + Vite (chat UI, releases, test results, integrations)
  - No — API-only, headless (use curl or external UI)

Question 6 - "Advanced Features"
  "Which advanced features do you want? (select all that apply)"
  Multi-select options:
  - AI triage (auto-classify test failures as real_bug / flaky / env_issue)
  - Release gates (automated go/no-go checks before deploy)
  - Hotfix detection (monitor alerts → auto-create tickets for critical issues)
  - Cron scheduler (periodic sync jobs, recurring checks)
  - Webhook triggers (GitHub, Jira, ClickUp events trigger bot actions)
```

After receiving answers, summarize the selections back to the user before generating code.

---

## Phase 2: Generate Project Structure

Create the target directory and generate the project skeleton.

### Always Generate

Read `references/tsconfig-template.json` and write it as `tsconfig.json`.

Read `references/package-template.json`. Customize it:
- Set the `name` field to the user's project name
- Remove `_comment_*` and `_conditional_dependencies` fields
- Include `openai` dependency if AI provider is OpenAI or Both
- Include `@anthropic-ai/sdk` if AI provider is Anthropic or Both
- Add runner-specific dependencies based on selections (e.g., `@playwright/test` for Playwright)
- Add client dependencies if dashboard selected (react, react-dom, vite, tailwindcss, etc.)

Write as `package.json`.

### Core Source Files

Always generate these files:

**`src/db.ts`** — PostgreSQL connection pool and migration runner:
```typescript
import pg from 'pg';
import fs from 'fs';
import path from 'path';
import dotenv from 'dotenv';
dotenv.config();

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

export async function query(text: string, params?: unknown[]) {
  return pool.query(text, params);
}

export async function migrate() {
  await query(`CREATE TABLE IF NOT EXISTS schema_migrations (
    id SERIAL PRIMARY KEY, filename VARCHAR(255) NOT NULL UNIQUE, applied_at TIMESTAMP DEFAULT NOW()
  )`);

  const dir = path.join(__dirname, 'migrations');
  const files = fs.readdirSync(dir).filter(f => f.endsWith('.sql')).sort();

  for (const file of files) {
    const { rows } = await query('SELECT 1 FROM schema_migrations WHERE filename = $1', [file]);
    if (rows.length > 0) continue;

    const sql = fs.readFileSync(path.join(dir, file), 'utf8');
    await query(sql);
    await query('INSERT INTO schema_migrations (filename) VALUES ($1)', [file]);
    console.log(`Applied migration: ${file}`);
  }
}

// Run migrations if called directly
if (process.argv[1]?.endsWith('db.ts') || process.argv[1]?.endsWith('db.js')) {
  migrate().then(() => { console.log('Migrations complete'); process.exit(0); })
    .catch(err => { console.error('Migration failed:', err); process.exit(1); });
}
```

**`src/events.ts`** — Event bus for WebSocket broadcasting:
```typescript
type Listener = (type: string, data: any) => void;
const listeners: Listener[] = [];

export function emit(type: string, data: any) {
  for (const fn of listeners) fn(type, data);
}

export function onBroadcast(fn: Listener) {
  listeners.push(fn);
}
```

**`src/auth.ts`** — Session-based authentication:
```typescript
import bcrypt from 'bcrypt';
import crypto from 'crypto';
import { query } from './db';

export async function createUser(email: string, password: string, name?: string) {
  const hash = await bcrypt.hash(password, 12);
  const { rows } = await query(
    'INSERT INTO users (email, password_hash, name) VALUES ($1, $2, $3) RETURNING id',
    [email, hash, name || null]
  );
  return rows[0].id;
}

export async function verifyPassword(email: string, password: string) {
  const { rows } = await query('SELECT * FROM users WHERE email = $1', [email]);
  if (!rows[0]) return null;
  const valid = await bcrypt.compare(password, rows[0].password_hash);
  return valid ? rows[0] : null;
}

export async function createSession(userId: number): Promise<string> {
  const id = crypto.randomBytes(32).toString('hex');
  const expires = new Date(Date.now() + 7 * 24 * 3600_000); // 7 days
  await query('INSERT INTO sessions (id, user_id, expires_at) VALUES ($1, $2, $3)', [id, userId, expires]);
  return id;
}

export async function getSession(sessionId: string) {
  const { rows } = await query(
    'SELECT s.*, u.email, u.name, u.role FROM sessions s JOIN users u ON s.user_id = u.id WHERE s.id = $1 AND s.expires_at > NOW()',
    [sessionId]
  );
  return rows[0] || null;
}
```

---

## Phase 3: Generate Bot Engine

Read `references/bot-engine-template.md` and generate the bot engine files.

### `src/bot/openai.ts` or `src/bot/anthropic.ts`

- If AI provider is **OpenAI** or **Both**: generate `src/bot/openai.ts` from the template
- If AI provider is **Anthropic** or **Both**: generate `src/bot/anthropic.ts` from the template
- If **Both**: generate both files and a `src/bot/ai.ts` wrapper that switches based on `process.env.AI_PROVIDER`

### `src/bot/engine.ts`

Generate from the template. **Customize the SYSTEM_PROMPT** to list only the integrations and runners the user selected. Example for a user who selected GitHub + Playwright + Slack:

```
Connected integrations and capabilities:
- GitHub: query PRs across repos, check releases, trigger CI workflows, view workflow runs
- Slack: send notifications for test results, deployments, bugs, releases
- Test runners: Playwright (web)
- Database: query releases, get test run stats, find untriaged failures
```

### `src/bot/tools.ts`

Generate from the template. **Include only tool definitions for selected services**:

- Always include: `get_releases`, `get_test_run_stats`, `get_untriaged_failures`
- If any runner selected: include `run_tests`
- Per integration, add the relevant tools from `references/base-integration-template.md`:
  - GitHub: `github_get_prs`, `github_get_release`, `github_trigger_workflow`, `github_get_workflow_runs`
  - ClickUp: `clickup_get_tasks`, `clickup_create_bug`, `clickup_sync_release_bugs`, `clickup_get_bug_count`
  - DataDog: `datadog_get_alerts`, `datadog_get_monitors`, `datadog_get_error_rate`, `datadog_get_latency`, `datadog_get_release_health`, `datadog_mute_monitor`
  - TestRail: `testrail_get_projects`, `testrail_get_runs`, `testrail_get_cases`, `testrail_create_run`, `testrail_add_result`
  - Slack: `slack_send_message`
  - Google Drive: `gdrive_get_release_folder`, `gdrive_upload_evidence`, `gdrive_create_folder`

### `src/bot/decisions.ts` (conditional)

If **AI triage**, **Release gates**, or **Hotfix detection** selected: read `references/decision-engine-template.md` and generate. Include only the sections the user selected.

### `src/bot/scheduler.ts` (conditional)

If **Cron scheduler** or **Webhook triggers** selected: read `references/scheduler-template.md` and generate. Populate `cronTasks` and `triggerRules` based on selected integrations.

---

## Phase 4: Generate Integration Adapters

Read `references/base-integration-template.md`.

### Always generate:
- `src/integrations/base.ts` — the abstract base class

### Per selection, generate the adapter:
- GitHub → `src/integrations/github.ts`
- ClickUp → `src/integrations/clickup.ts`
- Jira → `src/integrations/jira.ts` (stub with testConnection + basic methods)
- DataDog → `src/integrations/datadog.ts`
- Slack/Discord → `src/integrations/slack.ts`
- TestRail → `src/integrations/testrail.ts` (stub with testConnection + basic methods)
- Google Drive → `src/integrations/gdrive.ts` (stub with testConnection + basic methods)
- BrowserStack → referenced in runner configs, no separate adapter needed

### If Slack/Discord selected, also generate:
- `src/integrations/notifications.ts` — read from `references/notification-template.md`

---

## Phase 5: Generate Test Runners

Read `references/base-runner-template.md`.

### Always generate:
- `src/runners/base.ts` — abstract base class + interfaces
- `src/runners/index.ts` — platform-to-runner mapping (include only selected runners)

### Per selection:
- Playwright → `src/runners/playwright.ts`
- Maestro → `src/runners/maestro.ts`
- Appium → `src/runners/appium.ts` (stub following Maestro pattern)
- k6 → `src/runners/k6.ts`
- Visual regression → `src/runners/visual.ts`
- Custom → generate a `src/runners/custom.ts` skeleton with TODO comments

---

## Phase 6: Generate Database Migrations

Read `references/migration-templates.md`.

### Always generate:
- `src/migrations/001_initial.sql` — releases, bot_log, bot_state, integrations
- `src/migrations/002_auth.sql` — users, sessions
- `src/migrations/003_test_runs.sql` — test_runs, test_failures

### Conditionally generate:
- If **Webhook triggers** selected → `src/migrations/004_webhooks.sql`
- If **Cron scheduler** selected → `src/migrations/005_bot_jobs.sql`
- If **Release gates** selected → `src/migrations/006_validation_results.sql`

### Always generate seed migration:
- `src/migrations/099_seed_integrations.sql` — INSERT rows for each selected integration service

---

## Phase 7: Generate API Routes

### Always generate:

**`src/routes/bot.ts`** — Bot chat, history, status, start/stop, jobs endpoints.

**`src/routes/system.ts`** — Health check endpoint (`GET /api/health`).

**`src/routes/releases.ts`** — CRUD for releases (GET list, POST create, PATCH update phase/status).

**`src/routes/test-runs.ts`** — GET test runs with filters, GET failure details, POST trigger triage.

### Conditionally generate:

- If any integration selected → `src/routes/integrations.ts` (list status, test connection, save config)
- If **Webhook triggers** selected → `src/routes/webhooks.ts` (per-source webhook handlers). Read `references/scheduler-template.md` for the `processWebhookTrigger` import.
- If dashboard selected → `src/routes/auth.ts` (login, logout, session check)

### Always generate the main server:

**`src/index.ts`** — Express app with:
- dotenv config loading
- Rate limiting on `/api/*`
- JSON body parsing (1mb limit)
- Static file serving (from `public-v2/` if dashboard, else `public/`)
- All route mounts (only import the routes that were generated)
- WebSocket server on `/ws`
- Event bus → WebSocket broadcast wiring
- SPA fallback (if dashboard)
- Server listen on `process.env.PORT || 3000`

If **Cron scheduler** selected, import and call `startBot()` on server startup.

---

## Phase 8: Generate Dashboard (conditional)

**Only if the user selected "Yes" for dashboard.**

Read `references/dashboard-template.md` and generate the full client directory:

```
client/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── src/
    ├── main.tsx
    ├── index.css
    ├── App.tsx
    ├── components/
    │   └── Layout.tsx
    ├── lib/
    │   ├── api.ts
    │   └── ws.ts
    └── pages/
        ├── Overview.tsx
        ├── Bot.tsx
        ├── Releases.tsx
        ├── TestResults.tsx
        └── Integrations.tsx
```

---

## Phase 9: Environment and Finalization

### Generate `.env.example`

Read `references/env-template.md`. Include only the sections matching the user's selections.

### Generate `README.md`

Include:
1. Project name and one-line description
2. Architecture overview (which layers were generated)
3. Setup instructions:
   - `cp .env.example .env` and fill in values
   - Create PostgreSQL database
   - `npm install`
   - `npm run migrate`
   - `npm run dev`
   - If dashboard: `cd client && npm install && npm run dev`
4. Environment variables table (only the ones in `.env.example`)
5. API endpoint reference
6. How to add new integrations/runners (point to `/openclaw-qa-guide` skill)

### Print Setup Instructions

After all files are generated, print a summary:

```
Project generated at: ./[project-name]/

Files created: [count]
Integrations: [list]
Test runners: [list]
Features: [list]

Next steps:
1. cd [project-name]
2. cp .env.example .env  (fill in your credentials)
3. createdb openclaw_qa  (or your preferred database name)
4. npm install
5. npm run migrate
6. npm run dev

[If dashboard selected:]
7. cd client && npm install && npm run dev
   Dashboard: http://localhost:5173
   API: http://localhost:3000

For architecture docs and extension guides, use: /openclaw-qa-guide
```

---

## Important Rules

1. **No "Ecco" references anywhere** — use "your application" or "your product" in prompts and comments
2. **"OpenClaw" is the bot identity** — keep it in system prompts, UI labels, and log messages
3. **All URLs use `process.env.*`** — never hardcode domains
4. **All credentials come from DB or env** — never hardcode tokens or API keys
5. **Generate only what was selected** — don't include unused integrations, runners, or features
6. **Read reference files before generating** — use the templates in `references/` as the source of truth
7. **Preserve the architectural patterns** — BaseIntegration, BaseRunner, tool-calling loop, job queue
8. **TypeScript strict mode** — all generated code must be type-safe
