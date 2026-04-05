# Test Runner Templates

## Base Class: `src/runners/base.ts`

Always generate this file.

```typescript
import { query } from '../db';

export interface RunConfig {
  releaseId?: number;
  platform: string;       // web, ios, android, tv, api
  environment?: string;   // dev, staging, prod
  tags?: string[];        // filter which tests to run
  extraArgs?: string[];   // runner-specific CLI args
}

export interface RunResult {
  status: 'passed' | 'failed' | 'error';
  totalTests: number;
  passed: number;
  failed: number;
  skipped: number;
  durationMs: number;
  reportUrl?: string;
  errorSummary?: string;
  failures: FailureDetail[];
  metadata?: Record<string, unknown>;
}

export interface FailureDetail {
  testName: string;
  errorMessage: string;
  stackTrace?: string;
  screenshotUrl?: string;
  videoUrl?: string;
}

export abstract class BaseRunner {
  abstract readonly runner: string;

  async createRun(config: RunConfig): Promise<number> {
    const { rows } = await query(
      `INSERT INTO test_runs (release_id, runner, platform, status, started_at)
       VALUES ($1, $2, $3, 'running', NOW()) RETURNING id`,
      [config.releaseId || null, this.runner, config.platform]
    );
    return rows[0].id;
  }

  async completeRun(runId: number, result: RunResult): Promise<void> {
    await query(
      `UPDATE test_runs SET
        status = $1, total_tests = $2, passed = $3, failed = $4, skipped = $5,
        duration_ms = $6, report_url = $7, error_summary = $8, metadata = $9,
        finished_at = NOW()
       WHERE id = $10`,
      [result.status, result.totalTests, result.passed, result.failed, result.skipped,
       result.durationMs, result.reportUrl || null, result.errorSummary || null,
       JSON.stringify(result.metadata || {}), runId]
    );

    for (const f of result.failures) {
      await query(
        `INSERT INTO test_failures (test_run_id, test_name, error_message, stack_trace, screenshot_url, video_url)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [runId, f.testName, f.errorMessage, f.stackTrace || null, f.screenshotUrl || null, f.videoUrl || null]
      );
    }
  }

  async failRun(runId: number, error: string): Promise<void> {
    await query(
      `UPDATE test_runs SET status = 'error', error_summary = $1, finished_at = NOW() WHERE id = $2`,
      [error, runId]
    );
  }

  abstract execute(config: RunConfig): Promise<RunResult>;

  async run(config: RunConfig): Promise<{ runId: number; result: RunResult }> {
    const runId = await this.createRun(config);
    try {
      const result = await this.execute(config);
      await this.completeRun(runId, result);
      return { runId, result };
    } catch (err: any) {
      await this.failRun(runId, err.message || 'Unknown error');
      throw err;
    }
  }
}
```

## Runner Index: `src/runners/index.ts`

Always generate this file. Include only selected runners.

```typescript
import { BaseRunner, RunConfig, RunResult } from './base';
// Import only the runners the user selected:
// import { PlaywrightRunner } from './playwright';
// import { MaestroRunner } from './maestro';
// import { AppiumRunner } from './appium';
// import { K6Runner } from './k6';
// import { VisualRunner } from './visual';

export { BaseRunner, RunConfig, RunResult } from './base';

function runnerForPlatform(platform: string): BaseRunner {
  switch (platform) {
    // Uncomment based on selections:
    // case 'web': return new PlaywrightRunner();
    // case 'ios': case 'android': return new MaestroRunner();
    // case 'tv': return new AppiumRunner();
    // case 'api': return new K6Runner();
    // case 'visual': return new VisualRunner();
    default: throw new Error(`No runner configured for platform: ${platform}`);
  }
}

export async function runTestsForPlatform(
  platform: string,
  config: Partial<RunConfig> = {},
): Promise<{ runId: number; result: RunResult }> {
  const runner = runnerForPlatform(platform);
  return runner.run({ platform, ...config });
}
```

## Playwright Runner: `src/runners/playwright.ts`

Generate if Playwright selected.

```typescript
import { execSync } from 'child_process';
import { readFileSync, existsSync } from 'fs';
import { BaseRunner, RunConfig, RunResult, FailureDetail } from './base';

export class PlaywrightRunner extends BaseRunner {
  readonly runner = 'playwright';

  async execute(config: RunConfig): Promise<RunResult> {
    const baseUrl = this.getBaseUrl(config.environment);
    const reportPath = `/tmp/pw-report-${Date.now()}.json`;

    const args = [
      'npx', 'playwright', 'test',
      '--reporter=json',
    ];

    if (config.tags?.length) {
      args.push('--grep', config.tags.join('|'));
    }
    if (config.extraArgs?.length) {
      args.push(...config.extraArgs);
    }

    const startTime = Date.now();
    let stdout: string;

    try {
      stdout = execSync(args.join(' '), {
        encoding: 'utf8',
        timeout: 600_000,
        env: {
          ...process.env,
          BASE_URL: baseUrl,
          PLAYWRIGHT_JSON_OUTPUT_NAME: reportPath,
        },
      });
    } catch (err: any) {
      stdout = err.stdout || '';
      if (!stdout && !existsSync(reportPath)) {
        return {
          status: 'error',
          totalTests: 0, passed: 0, failed: 0, skipped: 0,
          durationMs: Date.now() - startTime,
          errorSummary: err.message,
          failures: [],
        };
      }
    }

    return this.parseReport(reportPath, stdout, Date.now() - startTime);
  }

  private getBaseUrl(environment?: string): string {
    if (!environment) return process.env.BASE_URL || 'http://localhost:3000';
    const key = `BASE_URL_${environment.toUpperCase()}`;
    return process.env[key] || process.env.BASE_URL || 'http://localhost:3000';
  }

  private parseReport(reportPath: string, stdout: string, durationMs: number): RunResult {
    let report: any;
    try {
      const raw = existsSync(reportPath) ? readFileSync(reportPath, 'utf8') : stdout;
      report = JSON.parse(raw);
    } catch {
      return {
        status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
        durationMs, errorSummary: 'Failed to parse Playwright JSON report', failures: [],
      };
    }

    const stats = report.stats || {};
    const failures: FailureDetail[] = [];

    for (const suite of report.suites || []) {
      this.extractFailures(suite, failures);
    }

    const total = (stats.expected || 0) + (stats.unexpected || 0) + (stats.skipped || 0);
    const failed = stats.unexpected || 0;
    const passed = stats.expected || 0;
    const skipped = stats.skipped || 0;

    return {
      status: failed > 0 ? 'failed' : 'passed',
      totalTests: total,
      passed, failed, skipped,
      durationMs: stats.duration || durationMs,
      failures,
    };
  }

  private extractFailures(suite: any, failures: FailureDetail[]): void {
    for (const spec of suite.specs || []) {
      for (const test of spec.tests || []) {
        if (test.status === 'unexpected') {
          const result = test.results?.[0];
          failures.push({
            testName: `${suite.title} > ${spec.title}`,
            errorMessage: result?.error?.message || 'Test failed',
            stackTrace: result?.error?.stack,
          });
        }
      }
    }
    for (const child of suite.suites || []) {
      this.extractFailures(child, failures);
    }
  }
}
```

## Maestro Runner: `src/runners/maestro.ts`

Generate if Maestro selected.

```typescript
import { execSync } from 'child_process';
import { readFileSync, existsSync, readdirSync } from 'fs';
import { BaseRunner, RunConfig, RunResult, FailureDetail } from './base';

export class MaestroRunner extends BaseRunner {
  readonly runner = 'maestro';

  async execute(config: RunConfig): Promise<RunResult> {
    const flowDir = config.tags?.[0] || './maestro-flows';
    const reportDir = `/tmp/maestro-report-${Date.now()}`;

    const args = ['maestro', 'test', flowDir, '--format', 'junit', '--output', reportDir];
    if (config.extraArgs?.length) args.push(...config.extraArgs);

    const startTime = Date.now();

    try {
      execSync(args.join(' '), { encoding: 'utf8', timeout: 600_000 });
    } catch (err: any) {
      if (!existsSync(reportDir)) {
        return {
          status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
          durationMs: Date.now() - startTime, errorSummary: err.message, failures: [],
        };
      }
    }

    return this.parseJUnitReport(reportDir, Date.now() - startTime);
  }

  private parseJUnitReport(reportDir: string, durationMs: number): RunResult {
    // Simplified JUnit XML parsing — in production, use a proper XML parser
    const files = existsSync(reportDir) ? readdirSync(reportDir).filter(f => f.endsWith('.xml')) : [];
    if (files.length === 0) {
      return { status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0, durationMs, errorSummary: 'No JUnit report found', failures: [] };
    }

    let total = 0, passed = 0, failed = 0, skipped = 0;
    const failures: FailureDetail[] = [];

    for (const file of files) {
      const xml = readFileSync(`${reportDir}/${file}`, 'utf8');
      const testsMatch = xml.match(/tests="(\d+)"/);
      const failsMatch = xml.match(/failures="(\d+)"/);
      const skipsMatch = xml.match(/skipped="(\d+)"/);

      const t = parseInt(testsMatch?.[1] || '0');
      const f = parseInt(failsMatch?.[1] || '0');
      const s = parseInt(skipsMatch?.[1] || '0');

      total += t;
      failed += f;
      skipped += s;
      passed += t - f - s;

      // Extract failure details (simplified regex parsing)
      const failureRegex = /<testcase name="([^"]*)"[^>]*>[\s\S]*?<failure[^>]*>([\s\S]*?)<\/failure>/g;
      let match;
      while ((match = failureRegex.exec(xml)) !== null) {
        failures.push({ testName: match[1], errorMessage: match[2].trim() });
      }
    }

    return {
      status: failed > 0 ? 'failed' : 'passed',
      totalTests: total, passed, failed, skipped,
      durationMs, failures,
    };
  }
}
```

## k6 Runner: `src/runners/k6.ts`

Generate if k6 selected.

```typescript
import { execSync } from 'child_process';
import { BaseRunner, RunConfig, RunResult } from './base';

export class K6Runner extends BaseRunner {
  readonly runner = 'k6';

  async execute(config: RunConfig): Promise<RunResult> {
    const scriptPath = config.tags?.[0] || './k6/load-test.js';
    const baseUrl = config.environment
      ? process.env[`BASE_URL_${config.environment.toUpperCase()}`]
      : process.env.BASE_URL || 'http://localhost:3000';

    const args = ['k6', 'run', '--out', 'json=/dev/stdout', '--summary-export=/dev/stderr', scriptPath];
    if (config.extraArgs?.length) args.push(...config.extraArgs);

    const startTime = Date.now();

    try {
      const output = execSync(args.join(' '), {
        encoding: 'utf8',
        timeout: 600_000,
        env: { ...process.env, K6_BASE_URL: baseUrl },
      });
      return this.parseSummary(output, Date.now() - startTime);
    } catch (err: any) {
      const output = err.stderr || err.stdout || '';
      if (output) return this.parseSummary(output, Date.now() - startTime);
      return {
        status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
        durationMs: Date.now() - startTime, errorSummary: err.message, failures: [],
      };
    }
  }

  private parseSummary(output: string, durationMs: number): RunResult {
    try {
      const summary = JSON.parse(output);
      const checks = summary.metrics?.checks?.values || {};
      const httpReqs = summary.metrics?.http_reqs?.values?.count || 0;
      const passes = Math.round(httpReqs * (checks.passes || 0) / Math.max(checks.passes + (checks.fails || 0), 1));
      const fails = httpReqs - passes;

      return {
        status: fails > 0 ? 'failed' : 'passed',
        totalTests: httpReqs,
        passed: passes,
        failed: fails,
        skipped: 0,
        durationMs,
        failures: [],
        metadata: {
          http_req_duration_avg: summary.metrics?.http_req_duration?.values?.avg,
          http_req_duration_p95: summary.metrics?.http_req_duration?.values?.['p(95)'],
          vus_max: summary.metrics?.vus_max?.values?.max,
        },
      };
    } catch {
      return {
        status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
        durationMs, errorSummary: 'Failed to parse k6 summary', failures: [],
      };
    }
  }
}
```

## Visual Regression Runner: `src/runners/visual.ts`

Generate if Visual Regression selected.

```typescript
import { execSync } from 'child_process';
import { BaseRunner, RunConfig, RunResult, FailureDetail } from './base';

export class VisualRunner extends BaseRunner {
  readonly runner = 'visual';

  async execute(config: RunConfig): Promise<RunResult> {
    const baseUrl = config.environment
      ? process.env[`BASE_URL_${config.environment.toUpperCase()}`]
      : process.env.BASE_URL || 'http://localhost:3000';

    // Uses Playwright for screenshot capture + pixel comparison
    const args = [
      'npx', 'playwright', 'test',
      '--project=visual',
      '--reporter=json',
    ];
    if (config.extraArgs?.length) args.push(...config.extraArgs);

    const startTime = Date.now();

    try {
      const stdout = execSync(args.join(' '), {
        encoding: 'utf8',
        timeout: 600_000,
        env: { ...process.env, BASE_URL: baseUrl },
      });
      return this.parseResults(stdout, Date.now() - startTime);
    } catch (err: any) {
      const stdout = err.stdout || '';
      if (stdout) return this.parseResults(stdout, Date.now() - startTime);
      return {
        status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
        durationMs: Date.now() - startTime, errorSummary: err.message, failures: [],
      };
    }
  }

  private parseResults(stdout: string, durationMs: number): RunResult {
    try {
      const report = JSON.parse(stdout);
      const stats = report.stats || {};
      const failures: FailureDetail[] = [];

      // Visual regression failures typically have screenshot diffs
      for (const suite of report.suites || []) {
        for (const spec of suite.specs || []) {
          for (const test of spec.tests || []) {
            if (test.status === 'unexpected') {
              const result = test.results?.[0];
              failures.push({
                testName: spec.title,
                errorMessage: result?.error?.message || 'Visual mismatch',
                screenshotUrl: result?.attachments?.find((a: any) => a.name === 'diff')?.path,
              });
            }
          }
        }
      }

      const total = (stats.expected || 0) + (stats.unexpected || 0);
      return {
        status: (stats.unexpected || 0) > 0 ? 'failed' : 'passed',
        totalTests: total,
        passed: stats.expected || 0,
        failed: stats.unexpected || 0,
        skipped: stats.skipped || 0,
        durationMs, failures,
      };
    } catch {
      return {
        status: 'error', totalTests: 0, passed: 0, failed: 0, skipped: 0,
        durationMs, errorSummary: 'Failed to parse visual test report', failures: [],
      };
    }
  }
}
```
