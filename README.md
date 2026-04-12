# @thinkscape/wreq-js

[![npm](https://img.shields.io/npm/v/%40thinkscape%2Fwreq-js.svg)](https://www.npmjs.com/package/@thinkscape/wreq-js)
[![CI](https://github.com/Thinkscape/wreq-js/actions/workflows/test.yml/badge.svg)](https://github.com/Thinkscape/wreq-js/actions/workflows/test.yml)

`@thinkscape/wreq-js` is Thinkscape's maintained fork of `wreq-js`: a Node.js and TypeScript HTTP client that helps you bypass TLS fingerprinting checks used by services like Cloudflare and DataDome, powered by native Rust bindings from [wreq](https://github.com/0x676e67/wreq).

If your requests work in a browser but get blocked from Node.js because your network fingerprint looks wrong, this is for you.
You keep a fetch style API and get browser profile level network behavior without running a full browser.

1. Built in browser TLS and HTTP fingerprint profiles across Chrome, Firefox, Safari, Edge, Opera, and OkHttp families
2. Native Rust engine for high throughput traffic with no browser process overhead
3. Fetch style API with sessions, cookies, proxies, and transport controls
4. WebSocket helper and constructor APIs with session cookie and transport reuse
5. TypeScript first developer experience with generated definitions
6. Native targets for macOS, Linux, and Windows

Common search terms: cloudflare bypass, datadome bypass, tls fingerprinting, ja3, ja4, browser impersonation, nodejs fetch, typescript http client.

## Used by this monorepo

In `agent-smart-fetch`, this fork powers the transport layer for:
- browser-profiled single fetches
- weighted request-phase progress events used by pi batch rendering
- attachment and binary downloads that are streamed to temp files by the shared smart-fetch core

The higher-level attachment/temp-file behavior lives in the smart-fetch packages in this monorepo, not in `@thinkscape/wreq-js` itself.

## Alternatives comparison

| Library | Approach | API | Notes |
|---------|----------|-----|-------|
| **wreq-js** | Rust native bindings ([wreq](https://github.com/0x676e67/wreq)) | Fetch style, TypeScript first | Profile labels and network behavior come from the native layer |
| [CycleTLS](https://github.com/Danny-Dasilva/CycleTLS) | Go subprocess bridge | Promise based | Subprocess model |
| [got-scraping](https://github.com/apify/got-scraping) | JavaScript HTTP client customization | `got` based | Header and request customization |
| [node-tls-client](https://github.com/Sahil1337/node-tls-client) | Native shared library bindings | Custom | Behavior depends on upstream native layer |
| [curl-impersonate](https://github.com/lwthiker/curl-impersonate) | curl based tooling | CLI and bindings | Binary/tooling workflow |

## Documentation

Documentation for this fork currently lives in the repository README and source tree while Thinkscape-specific docs are being stood up.

Upstream reference docs for the current API surface remain useful background material:
1. Quickstart: https://wreq.sqdsh.win/quickstart
2. API overview: https://wreq.sqdsh.win/api-reference/overview
3. Sessions: https://wreq.sqdsh.win/concepts/sessions
4. WebSockets: https://wreq.sqdsh.win/guides/websockets
5. Compatibility matrix: https://wreq.sqdsh.win/concepts/compatibility-matrix

## Installation

```bash
npm install @thinkscape/wreq-js
# or
yarn add @thinkscape/wreq-js
pnpm add @thinkscape/wreq-js
bun add @thinkscape/wreq-js
```

Current configured native target matrix in `package.json` includes:
1. macOS (Intel and Apple Silicon)
2. Linux (x64 glibc and musl, arm64 glibc)
3. Windows (x64)

If a matching prebuilt artifact is unavailable for your environment, installation may build from source (requires a Rust toolchain).

## Quick start

```ts
import { fetch } from '@thinkscape/wreq-js';

const res = await fetch('https://example.com/api', {
  browser: 'chrome_142',
  os: 'windows',
});

console.log(await res.json());
```

By default, standalone `fetch()` calls use isolated ephemeral cookie storage.
Use `createSession()` when you want cookie persistence across requests.

## Use sessions (recommended)

For **most real-world workloads**, start with a session and reuse it across requests.
This keeps one cookie and request context for multi step flows.

```ts
import { createSession } from '@thinkscape/wreq-js';

const session = await createSession({ browser: 'chrome_142', os: 'windows' });

try {
  const a = await session.fetch('https://example.com/a');
  const b = await session.fetch('https://example.com/b');
  console.log(a.status, b.status);
} finally {
  await session.close();
}
```

More session patterns: https://wreq.sqdsh.win

## WebSockets

Use the helper for a connected socket from one `await`.

```ts
import { websocket } from 'wreq-js';

const ws = await websocket('wss://example.com/ws', {
  browser: 'chrome_142',
  headers: {
    Authorization: 'Bearer token',
  },
});

ws.onmessage = (event) => {
  console.log(event.data);
};

ws.send('hello');
ws.close(1000, 'done');
```

Use the constructor when you want browser like `CONNECTING` behavior.

```ts
import { WebSocket } from 'wreq-js';

const ws = new WebSocket('wss://example.com/ws', {
  browser: 'chrome_142',
  os: 'windows',
});

ws.onopen = () => {
  void ws.send('connected');
};
```

Use `session.websocket(...)` to reuse cookies and transport settings from session HTTP calls.

```ts
import { createSession } from '@thinkscape/wreq-js';

const session = await createSession({ browser: 'chrome_142' });

try {
  await session.fetch('https://example.com/login', {
    method: 'POST',
    body: new URLSearchParams({ user: 'name', pass: 'secret' }),
  });

  const ws = await session.websocket('wss://example.com/ws');
  ws.onmessage = (event) => {
    console.log(event.data);
  };
} finally {
  await session.close();
}
```

## When to use

Use `wreq-js` when your Node.js HTTP or WebSocket traffic gets blocked because of TLS fingerprinting or browser profile mismatches.
It is a good fit when you want Cloudflare bypass and DataDome bypass style network behavior with a familiar fetch style API.
It handles transport and fingerprint level behavior, not CAPTCHA solving and not in page JavaScript execution.

If you need DOM/JS execution, CAPTCHA solving, or full browser automation, use Playwright/Puppeteer instead.

## FAQ

1. Why use sessions?
Use sessions for multi-step flows where cookie and request context should be shared.

2. Why does install compile from source on some machines?
If a matching prebuilt native artifact is unavailable, npm may build from source.

3. Can I use per-request proxy overrides inside a session?
Yes, by passing a `transport` on that specific `session.fetch(...)` call. The `proxy` field itself remains session-scoped.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Origins

This is a maintained fork of [will-work-for-meal/node-wreq](https://github.com/will-work-for-meal/node-wreq) (originally named `node-wreq`), with ongoing updates, compatibility fixes, and performance work.

## Acknowledgments

- [wreq](https://github.com/0x676e67/wreq) - Rust HTTP client with browser impersonation
- [wreq-util](https://github.com/0x676e67/wreq-util) - related browser profile tooling in the upstream ecosystem
- [NAPI-RS](https://napi.rs/) - Rust ↔ Node.js bindings
