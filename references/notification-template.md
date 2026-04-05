# Notification Router Template

## File: `src/integrations/notifications.ts`

Generate if Slack/Discord integration selected. Useful even without Slack for internal event routing.

```typescript
import { query } from '../db';

type EventType =
  | 'test_run_complete'
  | 'test_failure'
  | 'deploy_started'
  | 'deploy_success'
  | 'deploy_failed'
  | 'bug_created'
  | 'release_ready'
  | 'triage_complete'
  | 'datadog_alert';

interface NotificationRule {
  event: EventType;
  enabled: boolean;
  minSeverity?: 'low' | 'normal' | 'high' | 'critical';
  onlyOnFailure?: boolean;
}

interface NotificationPayload {
  event: EventType;
  release?: string;
  platform?: string;
  severity?: string;
  data: Record<string, any>;
}

const DEFAULT_RULES: NotificationRule[] = [
  { event: 'test_run_complete', enabled: true, onlyOnFailure: true },
  { event: 'test_failure', enabled: true },
  { event: 'deploy_started', enabled: true },
  { event: 'deploy_success', enabled: true },
  { event: 'deploy_failed', enabled: true },
  { event: 'bug_created', enabled: true },
  { event: 'release_ready', enabled: true },
  { event: 'triage_complete', enabled: true },
  { event: 'datadog_alert', enabled: true, minSeverity: 'high' },
];

const SEVERITY_ORDER: Record<string, number> = {
  low: 0, normal: 1, high: 2, critical: 3,
};

async function getRules(): Promise<NotificationRule[]> {
  const { rows } = await query(
    "SELECT config FROM integrations WHERE service = 'slack'"
  );
  const config = rows[0]?.config || {};
  if (config.notificationRules) {
    try { return JSON.parse(config.notificationRules); } catch { /* fall through */ }
  }
  return DEFAULT_RULES;
}

function shouldNotify(rule: NotificationRule, payload: NotificationPayload): boolean {
  if (!rule.enabled) return false;
  if (rule.onlyOnFailure && payload.data.failed === 0) return false;
  if (rule.minSeverity && payload.severity) {
    const minLevel = SEVERITY_ORDER[rule.minSeverity] ?? 0;
    const actualLevel = SEVERITY_ORDER[payload.severity] ?? 1;
    if (actualLevel < minLevel) return false;
  }
  return true;
}

export async function notify(payload: NotificationPayload): Promise<boolean> {
  const rules = await getRules();
  const rule = rules.find(r => r.event === payload.event);
  if (!rule || !shouldNotify(rule, payload)) return false;

  // Import your notification integration (Slack, Discord, etc.)
  // const { slack } = await import('./slack');

  const d = payload.data;

  switch (payload.event) {
    case 'test_run_complete':
      // return slack.notifyTestResults(payload.release, payload.platform, d.passed, d.failed, d.total, d.reportUrl);
      break;
    case 'deploy_started':
    case 'deploy_success':
    case 'deploy_failed':
      // const status = payload.event.replace('deploy_', '');
      // return slack.notifyDeployment(payload.release, payload.platform, d.environment, status);
      break;
    case 'bug_created':
      // return slack.notifyBugCreated(d.ticketId, d.summary, payload.release, d.priority);
      break;
    case 'release_ready':
      // return slack.notifyReleaseReady(payload.release, payload.platform, d.bugCount);
      break;
    case 'triage_complete':
      // return slack.notifyTriageResult(d.ticketId, d.verdict, d.summary);
      break;
    case 'datadog_alert':
      // return slack.notifyDataDogAlert(d.alertName, payload.release, payload.severity);
      break;
  }

  // Log the notification attempt
  await query('INSERT INTO bot_log (msg) VALUES ($1)', [
    `Notification: ${payload.event} for ${payload.release || 'global'}`,
  ]);

  return true;
}

export async function updateRules(rules: NotificationRule[]): Promise<void> {
  await query(
    `UPDATE integrations SET config = config || $1::jsonb WHERE service = 'slack'`,
    [JSON.stringify({ notificationRules: JSON.stringify(rules) })]
  );
}

export async function getNotificationRules(): Promise<NotificationRule[]> {
  return getRules();
}
```
