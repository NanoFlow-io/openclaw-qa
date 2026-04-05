# Dashboard Template

Generate only if user selected "Yes" for dashboard.

## Vite Config: `client/vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      '/api': 'http://localhost:3000',
      '/ws': { target: 'ws://localhost:3000', ws: true },
    },
  },
  build: {
    outDir: '../public-v2',
    emptyOutDir: true,
  },
});
```

## Client package.json: `client/package.json`

```json
{
  "name": "openclaw-qa-dashboard",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0",
    "lucide-react": "^1.7.0",
    "clsx": "^2.1.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "@tailwindcss/vite": "^4.0.0",
    "tailwindcss": "^4.0.0",
    "vite": "^8.0.0",
    "typescript": "~5.9.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  }
}
```

## Client tsconfig: `client/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

## CSS Entry: `client/src/index.css`

```css
@import "tailwindcss";
```

## App Entry: `client/src/main.tsx`

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

## App Router: `client/src/App.tsx`

```tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Layout from './components/Layout';
import Overview from './pages/Overview';
import Bot from './pages/Bot';
import Releases from './pages/Releases';
import TestResults from './pages/TestResults';
import Integrations from './pages/Integrations';

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<Navigate to="/overview" replace />} />
        <Route path="overview" element={<Overview />} />
        <Route path="bot" element={<Bot />} />
        <Route path="releases" element={<Releases />} />
        <Route path="test-results" element={<TestResults />} />
        <Route path="integrations" element={<Integrations />} />
      </Route>
    </Routes>
  );
}
```

## Layout: `client/src/components/Layout.tsx`

```tsx
import { Outlet, NavLink } from 'react-router-dom';
import { LayoutDashboard, Bot, Tag, TestTube, Plug } from 'lucide-react';
import clsx from 'clsx';

const nav = [
  { to: '/overview', label: 'Overview', icon: LayoutDashboard },
  { to: '/bot', label: 'OpenClaw', icon: Bot },
  { to: '/releases', label: 'Releases', icon: Tag },
  { to: '/test-results', label: 'Test Results', icon: TestTube },
  { to: '/integrations', label: 'Integrations', icon: Plug },
];

export default function Layout() {
  return (
    <div className="flex h-screen bg-gray-950 text-gray-100">
      <aside className="w-56 border-r border-gray-800 p-4 flex flex-col gap-1">
        <h1 className="text-lg font-bold mb-4 px-2">OpenClaw QA</h1>
        {nav.map(({ to, label, icon: Icon }) => (
          <NavLink key={to} to={to} className={({ isActive }) =>
            clsx('flex items-center gap-2 px-3 py-2 rounded text-sm',
              isActive ? 'bg-gray-800 text-white' : 'text-gray-400 hover:text-gray-200')
          }>
            <Icon size={16} /> {label}
          </NavLink>
        ))}
      </aside>
      <main className="flex-1 overflow-auto p-6">
        <Outlet />
      </main>
    </div>
  );
}
```

## API Client: `client/src/lib/api.ts`

```typescript
const BASE = '/api';

async function request<T>(path: string, opts?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...opts,
  });
  if (!res.ok) throw new Error(`API ${res.status}: ${await res.text()}`);
  return res.json();
}

export const api = {
  // Bot
  getBotStatus: () => request<any>('/bot'),
  chat: (message: string, context?: string) =>
    request<any>('/bot/chat', { method: 'POST', body: JSON.stringify({ message, context }) }),
  getHistory: () => request<any[]>('/bot/history'),
  clearChat: () => request<any>('/bot/clear', { method: 'POST' }),
  startBot: () => request<any>('/bot/start', { method: 'POST' }),
  stopBot: () => request<any>('/bot/stop', { method: 'POST' }),

  // Releases
  getReleases: () => request<any[]>('/releases'),
  createRelease: (data: any) => request<any>('/releases', { method: 'POST', body: JSON.stringify(data) }),

  // Test Runs
  getTestRuns: (limit?: number) => request<any[]>(`/test-runs?limit=${limit || 20}`),

  // Integrations
  getIntegrations: () => request<any[]>('/integrations'),
  testConnection: (service: string) => request<any>(`/integrations/${service}/test`, { method: 'POST' }),
  saveConfig: (service: string, config: any) =>
    request<any>(`/integrations/${service}/config`, { method: 'POST', body: JSON.stringify(config) }),
};
```

## WebSocket Hook: `client/src/lib/ws.ts`

```typescript
import { useEffect, useRef, useCallback } from 'react';

type WSHandler = (type: string, data: any) => void;

export function useWebSocket(handler: WSHandler) {
  const wsRef = useRef<WebSocket | null>(null);
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const ws = new WebSocket(`${protocol}//${window.location.host}/ws`);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      try {
        const { type, data } = JSON.parse(event.data);
        handlerRef.current(type, data);
      } catch { /* ignore malformed messages */ }
    };

    ws.onclose = () => {
      // Auto-reconnect after 3 seconds
      setTimeout(() => {
        if (wsRef.current === ws) {
          wsRef.current = null;
        }
      }, 3000);
    };

    return () => ws.close();
  }, []);
}
```

## Page Skeletons

### `client/src/pages/Overview.tsx`

```tsx
import { useEffect, useState } from 'react';
import { api } from '../lib/api';

export default function Overview() {
  const [releases, setReleases] = useState<any[]>([]);
  const [testRuns, setTestRuns] = useState<any[]>([]);

  useEffect(() => {
    api.getReleases().then(setReleases).catch(console.error);
    api.getTestRuns(5).then(setTestRuns).catch(console.error);
  }, []);

  return (
    <div>
      <h2 className="text-2xl font-bold mb-6">Dashboard</h2>
      <div className="grid grid-cols-3 gap-4 mb-8">
        <div className="bg-gray-900 rounded-lg p-4 border border-gray-800">
          <div className="text-sm text-gray-400">Active Releases</div>
          <div className="text-3xl font-bold">{releases.filter(r => r.status === 'active').length}</div>
        </div>
        <div className="bg-gray-900 rounded-lg p-4 border border-gray-800">
          <div className="text-sm text-gray-400">Recent Test Runs</div>
          <div className="text-3xl font-bold">{testRuns.length}</div>
        </div>
        <div className="bg-gray-900 rounded-lg p-4 border border-gray-800">
          <div className="text-sm text-gray-400">Open Bugs</div>
          <div className="text-3xl font-bold">{releases.reduce((s, r) => s + (r.bugs || 0), 0)}</div>
        </div>
      </div>
    </div>
  );
}
```

### `client/src/pages/Bot.tsx`

```tsx
import { useState, useRef, useEffect } from 'react';
import { Send, Trash2 } from 'lucide-react';
import { api } from '../lib/api';
import { useWebSocket } from '../lib/ws';

interface Message { role: string; content: string }
interface LogEntry { msg: string; created_at: string }

export default function Bot() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const [logs, setLogs] = useState<LogEntry[]>([]);
  const chatRef = useRef<HTMLDivElement>(null);

  useWebSocket((type, data) => {
    if (type === 'bot_log') {
      setLogs(prev => [data, ...prev].slice(0, 100));
    }
  });

  useEffect(() => {
    api.getHistory().then(setMessages).catch(console.error);
    api.getBotStatus().then(s => setLogs(s.log || [])).catch(console.error);
  }, []);

  useEffect(() => {
    chatRef.current?.scrollTo(0, chatRef.current.scrollHeight);
  }, [messages]);

  async function send() {
    if (!input.trim() || loading) return;
    const msg = input.trim();
    setInput('');
    setMessages(prev => [...prev, { role: 'user', content: msg }]);
    setLoading(true);
    try {
      const { reply } = await api.chat(msg);
      setMessages(prev => [...prev, { role: 'assistant', content: reply }]);
    } catch (err: any) {
      setMessages(prev => [...prev, { role: 'assistant', content: `Error: ${err.message}` }]);
    }
    setLoading(false);
  }

  return (
    <div className="grid grid-cols-2 gap-6">
      <div>
        <div className="flex items-center justify-between mb-4">
          <h2 className="text-xl font-bold">OpenClaw Chat</h2>
          <button onClick={() => { api.clearChat(); setMessages([]); }}
            className="text-gray-400 hover:text-white"><Trash2 size={16} /></button>
        </div>
        <div ref={chatRef} className="h-[500px] overflow-y-auto bg-gray-900 rounded-lg border border-gray-800 p-4 space-y-3 mb-3">
          {messages.map((m, i) => (
            <div key={i} className={m.role === 'user' ? 'text-right' : ''}>
              <span className={`inline-block px-3 py-2 rounded-lg text-sm ${
                m.role === 'user' ? 'bg-blue-600' : 'bg-gray-800'}`}>
                {m.content}
              </span>
            </div>
          ))}
          {loading && <div className="text-gray-500 text-sm">Thinking...</div>}
        </div>
        <div className="flex gap-2">
          <input value={input} onChange={e => setInput(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && send()}
            placeholder="Ask OpenClaw..." className="flex-1 bg-gray-900 border border-gray-700 rounded px-3 py-2 text-sm" />
          <button onClick={send} className="bg-blue-600 hover:bg-blue-500 px-4 py-2 rounded">
            <Send size={16} />
          </button>
        </div>
      </div>
      <div>
        <h3 className="text-xl font-bold mb-4">Activity Log</h3>
        <div className="h-[500px] overflow-y-auto bg-gray-900 rounded-lg border border-gray-800 p-4 space-y-2">
          {logs.map((log, i) => (
            <div key={i} className="text-xs text-gray-400">
              <span className="text-gray-600">{new Date(log.created_at).toLocaleTimeString()}</span>{' '}
              {log.msg}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

### Remaining Pages

Generate `Releases.tsx`, `TestResults.tsx`, and `Integrations.tsx` following the same pattern:

- **Releases**: Table of releases with tag, app, phase, bugs, status. Create release form.
- **TestResults**: Table of test runs with runner, platform, status, pass/fail counts. Click to expand failures.
- **Integrations**: Card per service showing status badge, test connection button, config form.

All pages use `api` client from `lib/api.ts` and the dark theme from Layout.

## HTML Entry: `client/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>OpenClaw QA</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```
