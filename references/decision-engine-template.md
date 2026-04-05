# Decision Engine Template

## File: `src/bot/decisions.ts`

Generate if AI Triage, Release Gates, or Hotfix Detection selected.

```typescript
import { query } from '../db';
import { chat, ChatMessage } from './openai'; // or './anthropic'

// =============================================
// TRIAGE — Generate if AI Triage selected
// =============================================

export type TriageVerdict = 'real_bug' | 'flaky' | 'env_issue';

interface TriageResult {
  failureId: number;
  verdict: TriageVerdict;
  confidence: number;
  reasoning: string;
}

export async function triageFailure(failureId: number): Promise<TriageResult> {
  const { rows: [failure] } = await query(
    `SELECT tf.*, tr.runner, tr.platform
     FROM test_failures tf JOIN test_runs tr ON tf.test_run_id = tr.id
     WHERE tf.id = $1`,
    [failureId]
  );
  if (!failure) throw new Error(`Failure ${failureId} not found`);

  // Check history for this test (flakiness indicator)
  const { rows: history } = await query(
    `SELECT triage_status, COUNT(*) as cnt
     FROM test_failures WHERE test_name = $1 AND id != $2
     GROUP BY triage_status`,
    [failure.test_name, failureId]
  );
  const historyStr = history.map((h: any) => `${h.triage_status}: ${h.cnt}`).join(', ') || 'no prior failures';

  const messages: ChatMessage[] = [
    {
      role: 'system',
      content: `You are a QA triage assistant. Classify test failures as one of: real_bug, flaky, env_issue.
Respond ONLY with valid JSON: {"verdict":"<verdict>","confidence":<0-100>,"reasoning":"<brief explanation>"}`,
    },
    {
      role: 'user',
      content: `Test failure to triage:
- Test: ${failure.test_name}
- Runner: ${failure.runner} | Platform: ${failure.platform}
- Error: ${failure.error_message || 'none'}
- Stack trace: ${(failure.stack_trace || 'none').slice(0, 500)}
- Prior history for this test: ${historyStr}

Classify this failure.`,
    },
  ];

  const result = await chat(messages);
  let parsed: { verdict: TriageVerdict; confidence: number; reasoning: string };

  try {
    parsed = JSON.parse(result.message);
  } catch {
    parsed = { verdict: 'real_bug', confidence: 50, reasoning: 'Could not parse AI response — defaulting to real_bug' };
  }

  await query('UPDATE test_failures SET triage_status = $1 WHERE id = $2', [parsed.verdict, failureId]);

  await query('INSERT INTO bot_log (msg) VALUES ($1)', [
    `Triage: failure #${failureId} "${failure.test_name}" -> ${parsed.verdict} (${parsed.confidence}%)`,
  ]);

  return { failureId, verdict: parsed.verdict, confidence: parsed.confidence, reasoning: parsed.reasoning };
}

export async function triageRun(testRunId: number): Promise<TriageResult[]> {
  const { rows } = await query(
    "SELECT id FROM test_failures WHERE test_run_id = $1 AND triage_status = 'untriaged'",
    [testRunId]
  );
  const results: TriageResult[] = [];
  for (const row of rows) {
    results.push(await triageFailure(row.id));
  }
  return results;
}

// =============================================
// RELEASE GATES — Generate if Release Gates selected
// =============================================

interface GateResult {
  releaseId: number;
  tag: string;
  passed: boolean;
  checks: { name: string; passed: boolean; detail: string }[];
}

export async function checkReleaseGate(releaseId: number): Promise<GateResult> {
  const { rows: [release] } = await query('SELECT * FROM releases WHERE id = $1', [releaseId]);
  if (!release) throw new Error(`Release ${releaseId} not found`);

  const checks: GateResult['checks'] = [];

  // Check 1: No open bugs
  const openBugs = release.bugs || 0;
  checks.push({
    name: 'bugs_to_zero',
    passed: openBugs === 0,
    detail: openBugs === 0 ? 'No open bugs' : `${openBugs} open bug(s) remaining`,
  });

  // Check 2: Latest test run passed
  const { rows: [latestRun] } = await query(
    `SELECT * FROM test_runs WHERE release_id = $1 ORDER BY id DESC LIMIT 1`,
    [releaseId]
  );
  const lastRunPassed = latestRun?.status === 'passed';
  checks.push({
    name: 'latest_test_passed',
    passed: lastRunPassed,
    detail: latestRun
      ? `Last run: ${latestRun.status} (${latestRun.passed}/${latestRun.total_tests} passed)`
      : 'No test runs found',
  });

  // Check 3: Coverage threshold (from test run metadata)
  const coverage = latestRun?.metadata?.coverage ?? null;
  if (coverage !== null) {
    checks.push({
      name: 'coverage_threshold',
      passed: coverage >= 80,
      detail: `Coverage: ${coverage}% (threshold: 80%)`,
    });
  }

  // Check 4: All failures triaged
  const { rows: [untriaged] } = await query(
    `SELECT COUNT(*) as cnt FROM test_failures tf
     JOIN test_runs tr ON tf.test_run_id = tr.id
     WHERE tr.release_id = $1 AND tf.triage_status = 'untriaged'`,
    [releaseId]
  );
  const untriagedCount = parseInt(untriaged.cnt);
  checks.push({
    name: 'all_triaged',
    passed: untriagedCount === 0,
    detail: untriagedCount === 0 ? 'All failures triaged' : `${untriagedCount} untriaged failure(s)`,
  });

  const passed = checks.every(c => c.passed);

  await query('INSERT INTO bot_log (msg) VALUES ($1)', [
    `Release gate: ${release.tag} — ${passed ? 'PASSED' : 'BLOCKED'} (${checks.filter(c => !c.passed).map(c => c.name).join(', ') || 'all clear'})`,
  ]);

  return { releaseId, tag: release.tag, passed, checks };
}

// =============================================
// HOTFIX DETECTION — Generate if Hotfix Detection selected
// =============================================

interface HotfixAlert {
  alertName: string;
  severity: string;
  release: string;
  ticketId: string | null;
}

export async function handleProductionAlert(
  alertName: string,
  severity: string,
  releaseTag: string,
): Promise<HotfixAlert> {
  const result: HotfixAlert = { alertName, severity, release: releaseTag, ticketId: null };

  // Only auto-assess critical/high severity
  if (severity !== 'critical' && severity !== 'high') {
    // Still notify, but don't auto-create tickets
    return result;
  }

  const messages: ChatMessage[] = [
    {
      role: 'system',
      content: `You assess production alerts to determine urgency. Respond with JSON: {"needsHotfix":true/false,"summary":"<brief description for ticket>","priority":1-4}`,
    },
    {
      role: 'user',
      content: `Production alert: "${alertName}" — Severity: ${severity} — Release: ${releaseTag}. Should this trigger a hotfix ticket?`,
    },
  ];

  const assessment = await chat(messages);
  let parsed: { needsHotfix: boolean; summary: string; priority: number };
  try {
    parsed = JSON.parse(assessment.message);
  } catch {
    parsed = { needsHotfix: severity === 'critical', summary: alertName, priority: severity === 'critical' ? 1 : 2 };
  }

  if (parsed.needsHotfix) {
    // Create a ticket in your task tracker integration
    // Example with ClickUp (adapt to your integration):
    // const { clickup } = await import('../integrations/clickup');
    // result.ticketId = await clickup.createTask(listId, {
    //   name: `[HOTFIX] ${parsed.summary}`,
    //   description: `Auto-created from production alert: ${alertName}\nSeverity: ${severity}\nRelease: ${releaseTag}`,
    //   priority: parsed.priority,
    //   tags: [releaseTag, 'hotfix'],
    // });

    await query('INSERT INTO bot_log (msg) VALUES ($1)', [
      `Hotfix: alert "${alertName}" assessed as needing hotfix (priority: ${parsed.priority})`,
    ]);
  }

  return result;
}
```
