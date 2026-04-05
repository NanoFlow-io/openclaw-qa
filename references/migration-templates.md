# Database Migration Templates

Generate migration SQL files based on user selections. All migrations use `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS` for idempotency.

## Always Generate: `001_initial.sql`

```sql
-- Core tables: releases, bot_log, bot_state, integrations

CREATE TABLE IF NOT EXISTS schema_migrations (
  id SERIAL PRIMARY KEY,
  filename VARCHAR(255) NOT NULL UNIQUE,
  applied_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS releases (
  id SERIAL PRIMARY KEY,
  tag VARCHAR(50) NOT NULL,
  app VARCHAR(50) NOT NULL,
  phase VARCHAR(20) NOT NULL DEFAULT 'Plan',
  bugs INTEGER NOT NULL DEFAULT 0,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS bot_log (
  id SERIAL PRIMARY KEY,
  msg TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS bot_state (
  id INTEGER PRIMARY KEY DEFAULT 1,
  status VARCHAR(20) NOT NULL DEFAULT 'stopped',
  last_run TIMESTAMP,
  last_action TEXT
);

INSERT INTO bot_state (id, status) VALUES (1, 'stopped') ON CONFLICT DO NOTHING;

CREATE TABLE IF NOT EXISTS integrations (
  service VARCHAR(50) PRIMARY KEY,
  status VARCHAR(20) NOT NULL DEFAULT 'not_configured',
  config JSONB NOT NULL DEFAULT '{}'
);
```

## Always Generate: `002_auth.sql`

```sql
-- Authentication tables

CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(100),
  role VARCHAR(20) DEFAULT 'user',
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sessions (
  id VARCHAR(64) PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_sessions_user ON sessions(user_id);
CREATE INDEX IF NOT EXISTS idx_sessions_expires ON sessions(expires_at);
```

## Always Generate: `003_test_runs.sql`

```sql
-- Test execution tracking

CREATE TABLE IF NOT EXISTS test_runs (
  id SERIAL PRIMARY KEY,
  release_id INTEGER REFERENCES releases(id) ON DELETE SET NULL,
  runner VARCHAR(30) NOT NULL,
  platform VARCHAR(20) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  total_tests INTEGER DEFAULT 0,
  passed INTEGER DEFAULT 0,
  failed INTEGER DEFAULT 0,
  skipped INTEGER DEFAULT 0,
  duration_ms INTEGER DEFAULT 0,
  report_url TEXT,
  error_summary TEXT,
  metadata JSONB DEFAULT '{}',
  started_at TIMESTAMP,
  finished_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_test_runs_release ON test_runs(release_id);
CREATE INDEX IF NOT EXISTS idx_test_runs_status ON test_runs(status);

CREATE TABLE IF NOT EXISTS test_failures (
  id SERIAL PRIMARY KEY,
  test_run_id INTEGER REFERENCES test_runs(id) ON DELETE CASCADE,
  test_name TEXT NOT NULL,
  error_message TEXT,
  stack_trace TEXT,
  screenshot_url TEXT,
  video_url TEXT,
  triage_status VARCHAR(20) DEFAULT 'untriaged',
  ticket_id VARCHAR(60),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_test_failures_run ON test_failures(test_run_id);
CREATE INDEX IF NOT EXISTS idx_test_failures_triage ON test_failures(triage_status);
```

## If Webhook Triggers Selected: `004_webhooks.sql`

```sql
-- Webhook event log

CREATE TABLE IF NOT EXISTS webhook_events (
  id SERIAL PRIMARY KEY,
  source VARCHAR(30) NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  processed BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_webhook_events_source ON webhook_events(source);
CREATE INDEX IF NOT EXISTS idx_webhook_events_created ON webhook_events(created_at);
```

## If Cron Scheduler Selected: `005_bot_jobs.sql`

```sql
-- Background job queue

CREATE TABLE IF NOT EXISTS bot_jobs (
  id SERIAL PRIMARY KEY,
  job_type VARCHAR(50) NOT NULL,
  release_id INTEGER REFERENCES releases(id) ON DELETE SET NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'queued',
  input JSONB DEFAULT '{}',
  output JSONB,
  error TEXT,
  scheduled_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  finished_at TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_bot_jobs_status ON bot_jobs(status);
CREATE INDEX IF NOT EXISTS idx_bot_jobs_scheduled ON bot_jobs(scheduled_at);
```

## If Release Gates Selected: `006_validation_results.sql`

```sql
-- Release gate validation results

CREATE TABLE IF NOT EXISTS validation_results (
  id SERIAL PRIMARY KEY,
  release_id INTEGER REFERENCES releases(id) ON DELETE CASCADE,
  gate_name VARCHAR(50) NOT NULL,
  passed BOOLEAN NOT NULL,
  checks JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_validation_release ON validation_results(release_id);
```

## Seed Integrations

Generate seed rows for each selected integration:

```sql
-- Seed integration rows (one per selected service)
INSERT INTO integrations (service, status, config) VALUES
  ('openai', 'not_configured', '{}'),      -- Always if OpenAI provider
  ('anthropic', 'not_configured', '{}'),    -- Always if Anthropic provider
  ('github', 'not_configured', '{}'),       -- If GitHub selected
  ('clickup', 'not_configured', '{}'),      -- If ClickUp selected
  ('jira', 'not_configured', '{}'),         -- If Jira selected
  ('datadog', 'not_configured', '{}'),      -- If DataDog selected
  ('slack', 'not_configured', '{}'),        -- If Slack/Discord selected
  ('discord', 'not_configured', '{}'),      -- If Slack/Discord selected
  ('testrail', 'not_configured', '{}'),     -- If TestRail selected
  ('gdrive', 'not_configured', '{}'),       -- If Google Drive selected
  ('browserstack', 'not_configured', '{}')  -- If BrowserStack selected
ON CONFLICT (service) DO NOTHING;
```
