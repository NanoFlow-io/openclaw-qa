# Integration Adapter Templates

## Base Class: `src/integrations/base.ts`

Always generate this file.

```typescript
import { query } from '../db';

export interface IntegrationConfig {
  [key: string]: string;
}

export interface IntegrationStatus {
  service: string;
  status: 'connected' | 'pending' | 'not_configured' | 'error';
  config: IntegrationConfig;
  lastChecked?: Date;
  error?: string;
}

export abstract class BaseIntegration {
  abstract readonly service: string;

  async getConfig(): Promise<IntegrationConfig> {
    const { rows } = await query('SELECT config FROM integrations WHERE service = $1', [this.service]);
    return rows[0]?.config || {};
  }

  async getStatus(): Promise<IntegrationStatus> {
    const { rows } = await query('SELECT * FROM integrations WHERE service = $1', [this.service]);
    const row = rows[0];
    return {
      service: this.service,
      status: row?.status || 'not_configured',
      config: row?.config || {},
    };
  }

  async saveConfig(config: IntegrationConfig, status: string = 'connected'): Promise<void> {
    await query(
      `UPDATE integrations SET config = $1, status = $2 WHERE service = $3`,
      [JSON.stringify(config), status, this.service]
    );
  }

  async setStatus(status: string, error?: string): Promise<void> {
    const configPatch = error ? { error } : {};
    await query(
      `UPDATE integrations SET status = $1, config = config || $2::jsonb WHERE service = $3`,
      [status, JSON.stringify(configPatch), this.service]
    );
  }

  abstract testConnection(): Promise<boolean>;
}
```

## GitHub Adapter: `src/integrations/github.ts`

Generate if GitHub integration selected.

```typescript
import { BaseIntegration } from './base';

class GitHubIntegration extends BaseIntegration {
  readonly service = 'github';

  private async request<T = any>(path: string, opts?: RequestInit): Promise<T> {
    const config = await this.getConfig();
    if (!config.token) throw new Error('GitHub token not configured');

    const res = await fetch(`https://api.github.com${path}`, {
      ...opts,
      headers: {
        Authorization: `token ${config.token}`,
        Accept: 'application/vnd.github.v3+json',
        ...opts?.headers,
      },
    });

    if (!res.ok) {
      const body = await res.text();
      throw new Error(`GitHub API ${res.status}: ${body}`);
    }
    return res.json() as T;
  }

  async testConnection(): Promise<boolean> {
    try {
      await this.request('/user');
      await this.setStatus('connected');
      return true;
    } catch (err: any) {
      await this.setStatus('error', err.message);
      return false;
    }
  }

  async getPRs(repo: string, state: string = 'open'): Promise<any[]> {
    const config = await this.getConfig();
    const owner = config.owner || '';
    const prs = await this.request(`/repos/${owner}/${repo}/pulls?state=${state}&per_page=30`);
    return prs.map((pr: any) => ({
      number: pr.number,
      title: pr.title,
      state: pr.state,
      author: pr.user?.login,
      url: pr.html_url,
      createdAt: pr.created_at,
      updatedAt: pr.updated_at,
    }));
  }

  async getRelease(repo: string, tag?: string): Promise<any> {
    const config = await this.getConfig();
    const owner = config.owner || '';
    const path = tag
      ? `/repos/${owner}/${repo}/releases/tags/${tag}`
      : `/repos/${owner}/${repo}/releases/latest`;
    return this.request(path);
  }

  async triggerWorkflow(repo: string, workflowId: string, ref: string = 'main', inputs?: Record<string, string>): Promise<void> {
    const config = await this.getConfig();
    const owner = config.owner || '';
    await this.request(`/repos/${owner}/${repo}/actions/workflows/${workflowId}/dispatches`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ref, inputs: inputs || {} }),
    });
  }

  async getWorkflowRuns(repo: string, limit: number = 10): Promise<any[]> {
    const config = await this.getConfig();
    const owner = config.owner || '';
    const data = await this.request(`/repos/${owner}/${repo}/actions/runs?per_page=${limit}`);
    return data.workflow_runs.map((r: any) => ({
      id: r.id,
      name: r.name,
      status: r.status,
      conclusion: r.conclusion,
      branch: r.head_branch,
      url: r.html_url,
      createdAt: r.created_at,
    }));
  }
}

export const github = new GitHubIntegration();
```

## ClickUp Adapter: `src/integrations/clickup.ts`

Generate if ClickUp integration selected.

```typescript
import { BaseIntegration } from './base';

class ClickUpIntegration extends BaseIntegration {
  readonly service = 'clickup';

  private async request<T = any>(path: string, opts?: RequestInit): Promise<T> {
    const config = await this.getConfig();
    if (!config.token) throw new Error('ClickUp token not configured');

    const res = await fetch(`https://api.clickup.com/api/v2${path}`, {
      ...opts,
      headers: {
        Authorization: config.token,
        'Content-Type': 'application/json',
        ...opts?.headers,
      },
    });

    if (!res.ok) throw new Error(`ClickUp API ${res.status}: ${await res.text()}`);
    return res.json() as T;
  }

  async testConnection(): Promise<boolean> {
    try {
      await this.request('/user');
      await this.setStatus('connected');
      return true;
    } catch (err: any) {
      await this.setStatus('error', err.message);
      return false;
    }
  }

  async getTasks(listId: string, tags?: string[]): Promise<any[]> {
    let path = `/list/${listId}/task?include_closed=true`;
    if (tags?.length) path += `&tags[]=${tags.join('&tags[]=')}`;
    const data = await this.request(path);
    return data.tasks.map((t: any) => ({
      id: t.id,
      name: t.name,
      status: t.status?.status,
      priority: t.priority?.priority,
      tags: t.tags?.map((tag: any) => tag.name) || [],
      assignees: t.assignees?.map((a: any) => a.username) || [],
      url: t.url,
    }));
  }

  async createTask(listId: string, task: {
    name: string;
    description?: string;
    priority?: number;
    tags?: string[];
    status?: string;
  }): Promise<string> {
    const data = await this.request(`/list/${listId}/task`, {
      method: 'POST',
      body: JSON.stringify(task),
    });
    return data.id;
  }
}

export const clickup = new ClickUpIntegration();
```

## DataDog Adapter: `src/integrations/datadog.ts`

Generate if DataDog integration selected.

```typescript
import { BaseIntegration } from './base';

class DataDogIntegration extends BaseIntegration {
  readonly service = 'datadog';

  private async request<T = any>(path: string): Promise<T> {
    const config = await this.getConfig();
    if (!config.apiKey || !config.appKey) throw new Error('DataDog keys not configured');

    const site = config.site || 'datadoghq.com';
    const res = await fetch(`https://api.${site}${path}`, {
      headers: {
        'DD-API-KEY': config.apiKey,
        'DD-APPLICATION-KEY': config.appKey,
      },
    });

    if (!res.ok) throw new Error(`DataDog API ${res.status}: ${await res.text()}`);
    return res.json() as T;
  }

  async testConnection(): Promise<boolean> {
    try {
      await this.request('/api/v1/validate');
      await this.setStatus('connected');
      return true;
    } catch (err: any) {
      await this.setStatus('error', err.message);
      return false;
    }
  }

  async getAlerts(): Promise<any[]> {
    const data = await this.request('/api/v1/monitor?monitor_tags=env:production&group_states=alert');
    return (data as any[]).filter((m: any) => m.overall_state === 'Alert').map((m: any) => ({
      id: m.id,
      name: m.name,
      type: m.type,
      state: m.overall_state,
      message: m.message,
    }));
  }

  async getErrorRate(service: string, minutes: number = 15): Promise<number> {
    const now = Math.floor(Date.now() / 1000);
    const from = now - minutes * 60;
    const query = `sum:trace.http.request.errors{service:${service}}.as_rate()`;
    const data = await this.request(`/api/v1/query?from=${from}&to=${now}&query=${encodeURIComponent(query)}`);
    const series = (data as any).series?.[0];
    if (!series?.pointlist?.length) return 0;
    const lastPoint = series.pointlist[series.pointlist.length - 1];
    return lastPoint[1] || 0;
  }
}

export const datadog = new DataDogIntegration();
```

## Slack Adapter: `src/integrations/slack.ts`

Generate if Slack/Discord integration selected.

```typescript
import { BaseIntegration } from './base';

class SlackIntegration extends BaseIntegration {
  readonly service = 'slack';

  private async send(text: string, blocks?: any[]): Promise<boolean> {
    const config = await this.getConfig();
    const url = config.webhookUrl;
    if (!url) return false;

    const body: any = { text };
    if (blocks) body.blocks = blocks;

    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    return res.ok;
  }

  async testConnection(): Promise<boolean> {
    try {
      const result = await this.send('OpenClaw QA connection test');
      if (result) await this.setStatus('connected');
      else await this.setStatus('error', 'Webhook returned non-OK');
      return result;
    } catch (err: any) {
      await this.setStatus('error', err.message);
      return false;
    }
  }

  async notifyTestResults(release: string, platform: string, passed: number, failed: number, total: number, reportUrl?: string): Promise<boolean> {
    const emoji = failed > 0 ? '🔴' : '🟢';
    const text = `${emoji} *Test Results* — ${release} (${platform})\nPassed: ${passed} | Failed: ${failed} | Total: ${total}${reportUrl ? `\n<${reportUrl}|View Report>` : ''}`;
    return this.send(text);
  }

  async notifyDeployment(release: string, platform: string, environment: string, status: 'started' | 'success' | 'failed'): Promise<boolean> {
    const emoji = { started: '🚀', success: '✅', failed: '❌' }[status];
    return this.send(`${emoji} *Deploy ${status}* — ${release} → ${environment} (${platform})`);
  }

  async notifyBugCreated(ticketId: string, summary: string, release: string, priority: string): Promise<boolean> {
    return this.send(`🐛 *Bug Created* — [${ticketId}] ${summary}\nRelease: ${release} | Priority: ${priority}`);
  }

  async notifyReleaseReady(release: string, platform: string, bugCount: number): Promise<boolean> {
    return this.send(`📦 *Release Ready* — ${release} (${platform}) | Open bugs: ${bugCount}`);
  }

  async notifyTriageResult(ticketId: string, verdict: string, summary: string): Promise<boolean> {
    const emoji = { real_bug: '🐛', flaky: '🔄', env_issue: '🌐' }[verdict] || '❓';
    return this.send(`${emoji} *Triage* — [${ticketId}] ${verdict}: ${summary}`);
  }

  async notifyDataDogAlert(alertName: string, release: string, severity: string): Promise<boolean> {
    const emoji = severity === 'critical' ? '🚨' : '⚠️';
    return this.send(`${emoji} *DataDog Alert* — ${alertName}\nRelease: ${release} | Severity: ${severity}`);
  }
}

export const slack = new SlackIntegration();
```

## Additional Adapters

For **Jira**, **TestRail**, **Google Drive**, and **BrowserStack** -- follow the same pattern:

1. Extend `BaseIntegration`
2. Implement `request()` helper with auth from `getConfig()`
3. Implement `testConnection()` that validates credentials
4. Add domain-specific methods
5. Export a singleton instance

The specific API calls vary by service but the structure is identical.
