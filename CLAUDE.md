---
description: Use Bun instead of Node.js, npm, pnpm, or vite.
globs: "*.ts, *.tsx, *.html, *.css, *.js, *.jsx, package.json"
alwaysApply: false
---

# claude-peers

Peer discovery and messaging MCP channel for Claude Code instances.

## Architecture

- `broker.ts` — HTTP daemon (default localhost:7899) + SQLite. Can bind to 0.0.0.0 for network access. Auto-launched by the MCP server in local mode.
- `server.ts` — MCP stdio server, one per Claude Code instance. Connects to broker (local or remote), exposes tools, pushes channel notifications.
- `shared/types.ts` — Shared TypeScript types for broker API.
- `shared/summarize.ts` — Auto-summary generation via gpt-5.4-nano.
- `cli.ts` — CLI utility for inspecting broker state.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `CLAUDE_PEERS_PORT` | `7899` | Broker HTTP port |
| `CLAUDE_PEERS_BIND` | `127.0.0.1` | Broker bind address. Set to `0.0.0.0` for network access |
| `CLAUDE_PEERS_HOST` | `127.0.0.1` | Broker hostname (used by server.ts and cli.ts to connect) |
| `CLAUDE_PEERS_REMOTE` | `false` | Set to `true` to skip auto-launching the broker (for remote connections) |
| `CLAUDE_PEERS_TOKEN` | _(empty)_ | Shared secret for bearer token auth. If set, all requests must include it |
| `CLAUDE_PEERS_DB` | `~/.claude-peers.db` | SQLite database path |
| `OPENAI_API_KEY` | _(empty)_ | Enables auto-summary via gpt-5.4-nano (optional) |

## Running Locally

```bash
# Start Claude Code with the channel:
claude --dangerously-load-development-channels server:claude-peers

# Or add to .mcp.json:
# { "claude-peers": { "command": "bun", "args": ["./server.ts"] } }

# CLI:
bun cli.ts status
bun cli.ts peers
bun cli.ts send <peer-id> <message>
bun cli.ts kill-broker
```

## Running Across the Network (e.g. Tailscale)

### 1. Start the broker on a central machine

```bash
CLAUDE_PEERS_BIND=0.0.0.0 CLAUDE_PEERS_TOKEN=your-secret bun broker.ts
```

### 2. Configure each Claude Code instance to connect remotely

Add to your `.mcp.json` (or `~/.claude/settings.json` under `mcpServers`):

```json
{
  "mcpServers": {
    "claude-peers": {
      "command": "bun",
      "args": ["/path/to/server.ts"],
      "env": {
        "CLAUDE_PEERS_HOST": "your-broker-host.tailnet.ts.net",
        "CLAUDE_PEERS_PORT": "7899",
        "CLAUDE_PEERS_REMOTE": "true",
        "CLAUDE_PEERS_TOKEN": "your-secret"
      }
    }
  }
}
```

### 3. Use the CLI from any machine

```bash
CLAUDE_PEERS_HOST=your-broker-host CLAUDE_PEERS_TOKEN=your-secret bun cli.ts status
```

### Notes

- Over Tailscale, WireGuard handles encryption, so plain HTTP is fine.
- The token is defense-in-depth since only your tailnet nodes can reach the broker.
- Local peers are cleaned up via PID checks; remote peers are cleaned up if they miss heartbeats for 60 seconds.
- Discovery scopes: `network` (all peers everywhere), `machine` (same hostname), `directory` (same cwd), `repo` (same git root).

## Bun

Default to using Bun instead of Node.js.

- Use `bun <file>` instead of `node <file>` or `ts-node <file>`
- Use `bun test` instead of `jest` or `vitest`
- Use `bun build <file.html|file.ts|file.css>` instead of `webpack` or `esbuild`
- Use `bun install` instead of `npm install` or `yarn install` or `pnpm install`
- Use `bun run <script>` instead of `npm run <script>` or `yarn run <script>` or `pnpm run <script>`
- Use `bunx <package> <command>` instead of `npx <package> <command>`
- Bun automatically loads .env, so don't use dotenv.

## APIs

- `Bun.serve()` supports WebSockets, HTTPS, and routes. Don't use `express`.
- `bun:sqlite` for SQLite. Don't use `better-sqlite3`.
- `Bun.redis` for Redis. Don't use `ioredis`.
- `Bun.sql` for Postgres. Don't use `pg` or `postgres.js`.
- `WebSocket` is built-in. Don't use `ws`.
- Prefer `Bun.file` over `node:fs`'s readFile/writeFile
- Bun.$`ls` instead of execa.

## Testing

Use `bun test` to run tests.

```ts#index.test.ts
import { test, expect } from "bun:test";

test("hello world", () => {
  expect(1).toBe(1);
});
```

## Frontend

Use HTML imports with `Bun.serve()`. Don't use `vite`. HTML imports fully support React, CSS, Tailwind.

Server:

```ts#index.ts
import index from "./index.html"

Bun.serve({
  routes: {
    "/": index,
    "/api/users/:id": {
      GET: (req) => {
        return new Response(JSON.stringify({ id: req.params.id }));
      },
    },
  },
  // optional websocket support
  websocket: {
    open: (ws) => {
      ws.send("Hello, world!");
    },
    message: (ws, message) => {
      ws.send(message);
    },
    close: (ws) => {
      // handle close
    }
  },
  development: {
    hmr: true,
    console: true,
  }
})
```

HTML files can import .tsx, .jsx or .js files directly and Bun's bundler will transpile & bundle automatically. `<link>` tags can point to stylesheets and Bun's CSS bundler will bundle.

```html#index.html
<html>
  <body>
    <h1>Hello, world!</h1>
    <script type="module" src="./frontend.tsx"></script>
  </body>
</html>
```

With the following `frontend.tsx`:

```tsx#frontend.tsx
import React from "react";
import { createRoot } from "react-dom/client";

// import .css files directly and it works
import './index.css';

const root = createRoot(document.body);

export default function Frontend() {
  return <h1>Hello, world!</h1>;
}

root.render(<Frontend />);
```

Then, run index.ts

```sh
bun --hot ./index.ts
```

For more information, read the Bun API docs in `node_modules/bun-types/docs/**.mdx`.
