# Bot Engine Templates

## OpenAI Client: `src/bot/openai.ts`

Generate if AI provider is OpenAI or Both.

```typescript
import OpenAI from 'openai';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export interface ChatMessage {
  role: 'system' | 'user' | 'assistant' | 'tool';
  content: string | null;
  tool_calls?: any[];
  tool_call_id?: string;
  name?: string;
}

export interface CompletionResult {
  message: string;
  toolCalls: OpenAI.Chat.Completions.ChatCompletionMessageToolCall[];
}

export async function chat(
  messages: ChatMessage[],
  tools?: OpenAI.Chat.Completions.ChatCompletionTool[],
  fast?: boolean,
): Promise<CompletionResult> {
  const model = fast
    ? (process.env.OPENAI_MODEL || 'gpt-4o-mini')
    : (process.env.OPENAI_MODEL_HOTFIX || process.env.OPENAI_MODEL || 'gpt-4o-mini');

  const response = await client.chat.completions.create({
    model,
    messages: messages as any,
    tools: tools?.length ? tools : undefined,
    tool_choice: tools?.length ? 'auto' : undefined,
  });

  const choice = response.choices[0];
  return {
    message: choice.message.content || '',
    toolCalls: choice.message.tool_calls || [],
  };
}
```

## Anthropic Client: `src/bot/anthropic.ts`

Generate if AI provider is Anthropic or Both.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export interface ChatMessage {
  role: 'system' | 'user' | 'assistant' | 'tool';
  content: string | null;
  tool_calls?: any[];
  tool_call_id?: string;
  name?: string;
}

export interface CompletionResult {
  message: string;
  toolCalls: { id: string; function: { name: string; arguments: string } }[];
}

// Convert OpenAI-style tool definitions to Anthropic format
function convertTools(tools: any[]): Anthropic.Tool[] {
  return tools.map(t => ({
    name: t.function.name,
    description: t.function.description,
    input_schema: t.function.parameters,
  }));
}

// Convert OpenAI-style messages to Anthropic format
function convertMessages(messages: ChatMessage[]): { system: string; messages: Anthropic.MessageParam[] } {
  const system = messages.find(m => m.role === 'system')?.content || '';
  const rest = messages.filter(m => m.role !== 'system');

  const converted: Anthropic.MessageParam[] = [];
  for (const msg of rest) {
    if (msg.role === 'user') {
      converted.push({ role: 'user', content: msg.content || '' });
    } else if (msg.role === 'assistant') {
      if (msg.tool_calls?.length) {
        converted.push({
          role: 'assistant',
          content: msg.tool_calls.map(tc => ({
            type: 'tool_use' as const,
            id: tc.id,
            name: tc.function.name,
            input: JSON.parse(tc.function.arguments),
          })),
        });
      } else {
        converted.push({ role: 'assistant', content: msg.content || '' });
      }
    } else if (msg.role === 'tool') {
      converted.push({
        role: 'user',
        content: [{
          type: 'tool_result' as const,
          tool_use_id: msg.tool_call_id!,
          content: msg.content || '',
        }],
      });
    }
  }

  return { system, messages: converted };
}

export async function chat(
  messages: ChatMessage[],
  tools?: any[],
  fast?: boolean,
): Promise<CompletionResult> {
  const model = process.env.ANTHROPIC_MODEL || 'claude-sonnet-4-6';
  const { system, messages: convertedMessages } = convertMessages(messages);

  const response = await client.messages.create({
    model,
    max_tokens: 4096,
    system,
    messages: convertedMessages,
    tools: tools?.length ? convertTools(tools) : undefined,
  });

  const textBlocks = response.content.filter(b => b.type === 'text');
  const toolBlocks = response.content.filter(b => b.type === 'tool_use');

  return {
    message: textBlocks.map(b => b.text).join('\n'),
    toolCalls: toolBlocks.map(b => ({
      id: b.id,
      function: { name: b.name, arguments: JSON.stringify(b.input) },
    })),
  };
}
```

## Engine: `src/bot/engine.ts`

Always generate this file. The system prompt should list only the connected integrations and runners.

```typescript
import { query } from '../db';
import { chat, ChatMessage, CompletionResult } from './openai'; // or './anthropic'
import { toolDefinitions, executeTool } from './tools';

const SYSTEM_PROMPT = `You are OpenClaw, an AI release operations assistant for your application.
Your job is to orchestrate the release lifecycle: manage tickets, trigger tests,
monitor production health, triage test failures, and coordinate the team.

You have tools to interact with all connected integrations. Always be concise and action-oriented.
When you take an action, confirm what you did. If you need more information, ask specific questions.

Rules:
- When asked to run tests, always proceed immediately. Do not ask for confirmation.
- When asked to check alerts or triage, use the tools right away.
- Be action-oriented: execute first, report results after.

Connected integrations and capabilities:
{{CAPABILITIES_LIST}}`;
// NOTE: Replace {{CAPABILITIES_LIST}} with the actual integrations and runners
// selected during setup. Example:
// - GitHub: query PRs, check releases, trigger CI workflows
// - Playwright: run web tests
// - DataDog: check alerts, query metrics

const conversations = new Map<string, ChatMessage[]>();
const MAX_HISTORY = 40;

function getConversation(context: string): ChatMessage[] {
  if (!conversations.has(context)) {
    conversations.set(context, [
      { role: 'system', content: SYSTEM_PROMPT },
    ]);
  }
  return conversations.get(context)!;
}

function trimHistory(messages: ChatMessage[]): void {
  if (messages.length > MAX_HISTORY + 1) {
    const system = messages[0];
    const recent = messages.slice(-(MAX_HISTORY));
    messages.length = 0;
    messages.push(system, ...recent);
  }
}

export async function processMessage(
  userMessage: string,
  context: string = 'global',
): Promise<{ reply: string; toolsUsed: string[] }> {
  const messages = getConversation(context);
  messages.push({ role: 'user', content: userMessage });

  const toolsUsed: string[] = [];
  let iterations = 0;
  const maxIterations = 10;

  while (iterations < maxIterations) {
    iterations++;

    const result = await chat(messages, toolDefinitions);

    if (result.toolCalls.length === 0) {
      messages.push({ role: 'assistant', content: result.message });
      trimHistory(messages);

      await query('INSERT INTO bot_log (msg) VALUES ($1)', [
        `OpenClaw: ${result.message.slice(0, 200)}`,
      ]);

      return { reply: result.message, toolsUsed };
    }

    messages.push({
      role: 'assistant',
      content: result.message || null,
      tool_calls: result.toolCalls.map(tc => ({
        id: tc.id,
        type: 'function' as const,
        function: { name: tc.function.name, arguments: tc.function.arguments },
      })),
    } as any);

    for (const tc of result.toolCalls) {
      const toolName = tc.function.name;
      toolsUsed.push(toolName);

      let args: Record<string, any>;
      try {
        args = JSON.parse(tc.function.arguments);
      } catch {
        args = {};
      }

      await query('INSERT INTO bot_log (msg) VALUES ($1)', [
        `OpenClaw tool: ${toolName}(${JSON.stringify(args).slice(0, 150)})`,
      ]);

      const toolResult = await executeTool(toolName, args);

      messages.push({
        role: 'tool',
        content: toolResult,
        tool_call_id: tc.id,
        name: toolName,
      });
    }
  }

  const fallback = 'I reached the maximum number of tool calls. Here is what I have so far.';
  messages.push({ role: 'assistant', content: fallback });
  return { reply: fallback, toolsUsed };
}

export function clearConversation(context: string = 'global'): void {
  conversations.delete(context);
}

export function getHistory(context: string = 'global'): ChatMessage[] {
  return getConversation(context).filter(m => m.role !== 'system');
}

export async function runAction(action: string): Promise<string> {
  const { reply } = await processMessage(action);
  return reply;
}
```

## Tool Registry: `src/bot/tools.ts`

Always generate. Include only tool definitions for selected integrations and runners.

```typescript
import { query } from '../db';
import type OpenAI from 'openai';

type ToolDef = OpenAI.Chat.Completions.ChatCompletionTool;

export const toolDefinitions: ToolDef[] = [
  // === Always include: database queries ===
  {
    type: 'function',
    function: {
      name: 'get_releases',
      description: 'Get active releases from the database',
      parameters: {
        type: 'object',
        properties: {
          status: { type: 'string', description: 'Filter by status (active, completed)' },
        },
      },
    },
  },
  {
    type: 'function',
    function: {
      name: 'get_test_run_stats',
      description: 'Get test run statistics for a release',
      parameters: {
        type: 'object',
        properties: {
          releaseId: { type: 'number', description: 'Release ID' },
        },
        required: ['releaseId'],
      },
    },
  },

  // === Always include if any runner selected: run_tests ===
  {
    type: 'function',
    function: {
      name: 'run_tests',
      description: 'Run tests for a given platform',
      parameters: {
        type: 'object',
        properties: {
          platform: { type: 'string', description: 'Platform: web, ios, android, tv, api, visual' },
          environment: { type: 'string', description: 'Target environment: dev, staging, prod' },
          tags: { type: 'array', items: { type: 'string' }, description: 'Tags or spec filters' },
        },
        required: ['platform'],
      },
    },
  },

  // === Add per-integration tool definitions below ===
  // See base-integration-template.md for examples per service
];

async function executeToolInner(name: string, args: Record<string, any>): Promise<any> {
  switch (name) {
    // --- Database tools (always) ---
    case 'get_releases': {
      const status = args.status || 'active';
      const { rows } = await query('SELECT * FROM releases WHERE status = $1 ORDER BY id DESC LIMIT 20', [status]);
      return rows;
    }
    case 'get_test_run_stats': {
      const { rows } = await query(
        `SELECT runner, platform, status, SUM(passed) as passed, SUM(failed) as failed, COUNT(*) as runs
         FROM test_runs WHERE release_id = $1 GROUP BY runner, platform, status`,
        [args.releaseId]
      );
      return rows;
    }

    // --- Test runner (always if runners selected) ---
    case 'run_tests': {
      const { runTestsForPlatform } = await import('../runners');
      const { runId, result } = await runTestsForPlatform(args.platform, {
        environment: args.environment,
        tags: args.tags,
      });
      return { runId, ...result };
    }

    // === Add per-integration executors below ===
    // Example for GitHub:
    // case 'github_get_prs': {
    //   const { github } = await import('../integrations/github');
    //   return github.getPRs(args.repo, args.state);
    // }

    default:
      return { error: `Unknown tool: ${name}` };
  }
}

export async function executeTool(name: string, args: Record<string, any>): Promise<string> {
  try {
    const result = await executeToolInner(name, args);
    return JSON.stringify(result);
  } catch (err: any) {
    return JSON.stringify({ error: err.message });
  }
}
```
