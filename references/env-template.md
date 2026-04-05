# Environment Variables Template

Generate a `.env.example` file containing only the sections relevant to the user's selections.

## Always Include

```env
# --- Server ---
PORT=3000
NODE_ENV=development

# --- Database ---
DATABASE_URL=postgresql://user:password@localhost:5432/openclaw_qa

# --- Session ---
SESSION_SECRET=change-me-to-a-random-string
```

## If AI Provider = OpenAI or Both

```env
# --- OpenAI ---
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
OPENAI_MODEL_HOTFIX=gpt-4o
```

## If AI Provider = Anthropic or Both

```env
# --- Anthropic ---
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-4-6
```

## If GitHub Integration Selected

```env
# --- GitHub ---
GITHUB_TOKEN=ghp_...
GITHUB_OWNER=your-org
GITHUB_REPOS=repo-one,repo-two
GITHUB_WEBHOOK_SECRET=your-webhook-secret
```

## If ClickUp Integration Selected

```env
# --- ClickUp ---
CLICKUP_TOKEN=pk_...
CLICKUP_LIST_IDS={"Web":"123","Mobile":"456","Backend":"789"}
```

## If Jira Integration Selected

```env
# --- Jira ---
JIRA_BASE_URL=https://your-org.atlassian.net
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=...
JIRA_PROJECT_KEY=QA
```

## If DataDog Integration Selected

```env
# --- DataDog ---
DATADOG_API_KEY=...
DATADOG_APP_KEY=...
DATADOG_SITE=datadoghq.com
```

## If Slack/Discord Integration Selected

```env
# --- Slack / Discord ---
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...
```

## If TestRail Integration Selected

```env
# --- TestRail ---
TESTRAIL_BASE_URL=https://your-org.testrail.io
TESTRAIL_USER=you@company.com
TESTRAIL_API_KEY=...
```

## If Google Drive Integration Selected

```env
# --- Google Drive ---
GDRIVE_SERVICE_ACCOUNT_KEY=./credentials/gdrive-sa.json
GDRIVE_ROOT_FOLDER_ID=...
```

## If BrowserStack Integration Selected

```env
# --- BrowserStack ---
BROWSERSTACK_USER=your-username
BROWSERSTACK_KEY=your-access-key
```

## If Test Runners Selected

```env
# --- Test Runner Base URLs ---
BASE_URL=http://localhost:3000
BASE_URL_DEV=https://dev.your-app.com
BASE_URL_STAGING=https://staging.your-app.com
BASE_URL_PROD=https://your-app.com
```
