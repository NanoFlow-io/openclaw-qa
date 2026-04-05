# Scheduler & Job Queue Template

## File: `src/bot/scheduler.ts`

Generate if Cron Scheduler or Webhook Triggers selected.

```typescript
import { query } from '../db';
import { processMessage } from './engine';

// --- Job Queue Processor ---

interface BotJob {
  id: number;
  job_type: string;
  release_id: number | null;
  status: string;
  input: Record<string, any>;
}

let polling = false;
let pollTimer: ReturnType<typeof setInterval> | null = null;

async function claimNextJob(): Promise<BotJob | null> {
  const { rows } = await query(
    `UPDATE bot_jobs SET status = 'running', started_at = NOW()
     WHERE id = (
       SELECT id FROM bot_jobs
       WHERE status = 'queued' AND scheduled_at <= NOW()
       ORDER BY scheduled_at ASC LIMIT 1
       FOR UPDATE SKIP LOCKED
     ) RETURNING *`
  );
  return rows[0] || null;
}

async function completeJob(jobId: number, output: any): Promise<void> {
  await query(
    `UPDATE bot_jobs SET status = 'completed', output = $1::jsonb, finished_at = NOW() WHERE id = $2`,
    [JSON.stringify(output), jobId]
  );
}

async function failJob(jobId: number, error: string): Promise<void> {
  await query(
    `UPDATE bot_jobs SET status = 'failed', error = $1, finished_at = NOW() WHERE id = $2`,
    [error, jobId]
  );
}

async function executeJob(job: BotJob): Promise<void> {
  try {
    let output: any;

    switch (job.job_type) {
      case 'chat': {
        const context = job.input.context || 'global';
        const { reply, toolsUsed } = await processMessage(job.input.message, context);
        output = { reply, toolsUsed };
        break;
      }
      case 'run_tests': {
        const { runTestsForPlatform } = await import('../runners');
        output = await runTestsForPlatform(job.input.platform, {
          tags: job.input.releaseTag ? [job.input.releaseTag] : undefined,
        });
        break;
      }
      case 'notify': {
        const { notify } = await import('../integrations/notifications');
        const sent = await notify(job.input as any);
        output = { sent };
        break;
      }
      // Add custom job types here based on selected integrations:
      // case 'sync_tasks': { ... }
      default: {
        const { reply, toolsUsed } = await processMessage(
          `Execute job: ${job.job_type} with params: ${JSON.stringify(job.input)}`
        );
        output = { reply, toolsUsed };
      }
    }

    await completeJob(job.id, output);
    await query('INSERT INTO bot_log (msg) VALUES ($1)', [
      `Job ${job.id} (${job.job_type}) completed`,
    ]);
  } catch (err: any) {
    await failJob(job.id, err.message);
    await query('INSERT INTO bot_log (msg) VALUES ($1)', [
      `Job ${job.id} (${job.job_type}) failed: ${err.message}`,
    ]);
  }
}

async function pollOnce(): Promise<void> {
  const job = await claimNextJob();
  if (job) await executeJob(job);
}

export function startJobPoller(intervalMs: number = 5000): void {
  if (polling) return;
  polling = true;
  pollTimer = setInterval(async () => {
    try { await pollOnce(); } catch (err: any) { console.error('Job poller error:', err.message); }
  }, intervalMs);
}

export function stopJobPoller(): void {
  polling = false;
  if (pollTimer) { clearInterval(pollTimer); pollTimer = null; }
}

// --- Cron Scheduler ---

interface CronTask {
  name: string;
  cronExpr: string;   // 'every:Xm' or 'every:Xh'
  jobType: string;
  input: Record<string, any>;
}

// Configure your recurring tasks here:
const cronTasks: CronTask[] = [
  // Example: sync tasks every 15 minutes
  // { name: 'sync-tasks', cronExpr: 'every:15m', jobType: 'sync_tasks', input: {} },
  // Example: check alerts every 10 minutes
  // { name: 'check-alerts', cronExpr: 'every:10m', jobType: 'chat', input: { message: 'Check for critical alerts', context: 'global' } },
];

const cronTimers: Map<string, ReturnType<typeof setInterval>> = new Map();

function parseCronInterval(expr: string): number {
  const match = expr.match(/^every:(\d+)(m|h)$/);
  if (!match) return 15 * 60_000;
  const val = parseInt(match[1]);
  return match[2] === 'h' ? val * 3600_000 : val * 60_000;
}

async function enqueueCronJob(task: CronTask): Promise<void> {
  await query(
    `INSERT INTO bot_jobs (job_type, input, scheduled_at) VALUES ($1, $2::jsonb, NOW())`,
    [task.jobType, JSON.stringify(task.input)]
  );
  await query('INSERT INTO bot_log (msg) VALUES ($1)', [
    `Cron: enqueued ${task.name} (${task.jobType})`,
  ]);
}

export function startCronScheduler(): void {
  for (const task of cronTasks) {
    const interval = parseCronInterval(task.cronExpr);
    const timer = setInterval(() => enqueueCronJob(task).catch(console.error), interval);
    cronTimers.set(task.name, timer);
  }
}

export function stopCronScheduler(): void {
  for (const [, timer] of cronTimers) clearInterval(timer);
  cronTimers.clear();
}

export function addCronTask(task: CronTask): void {
  cronTasks.push(task);
  const interval = parseCronInterval(task.cronExpr);
  const timer = setInterval(() => enqueueCronJob(task).catch(console.error), interval);
  cronTimers.set(task.name, timer);
}

// --- Event-Driven Triggers ---

interface TriggerRule {
  source: string;
  eventType: string;
  jobType: string;
  input: (payload: any) => Record<string, any>;
}

// Configure webhook-to-job trigger rules:
const triggerRules: TriggerRule[] = [
  // Example: GitHub PR events trigger a chat message
  // {
  //   source: 'github', eventType: 'pull_request', jobType: 'chat',
  //   input: (p) => ({ message: `PR ${p.action}: "${p.pull_request?.title}" on ${p.repository?.name}`, context: 'global' }),
  // },
];

export async function processWebhookTrigger(source: string, eventType: string, payload: any): Promise<boolean> {
  const matching = triggerRules.filter(r => r.source === source && r.eventType === eventType);
  if (matching.length === 0) return false;

  for (const rule of matching) {
    const input = rule.input(payload);
    await query(
      `INSERT INTO bot_jobs (job_type, input, scheduled_at) VALUES ($1, $2::jsonb, NOW())`,
      [rule.jobType, JSON.stringify(input)]
    );
    await query('INSERT INTO bot_log (msg) VALUES ($1)', [
      `Trigger: ${source}/${eventType} -> ${rule.jobType}`,
    ]);
  }

  return true;
}

// --- Master start/stop ---

export function startBot(): void {
  startJobPoller();
  startCronScheduler();
}

export function stopBot(): void {
  stopJobPoller();
  stopCronScheduler();
}

export async function enqueueJob(jobType: string, input: Record<string, any>, releaseId?: number): Promise<number> {
  const { rows: [job] } = await query(
    `INSERT INTO bot_jobs (job_type, release_id, input, scheduled_at) VALUES ($1, $2, $3::jsonb, NOW()) RETURNING id`,
    [jobType, releaseId || null, JSON.stringify(input)]
  );
  return job.id;
}
```
