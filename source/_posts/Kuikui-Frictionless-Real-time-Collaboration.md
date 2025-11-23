---
title: "Kuikui: Frictionless Real-time Collaboration"
date: 2025-11-23
categories:
  - Projects
tags:
  - TypeScript
  - Tooling
  - Socket
---

## The Philosophy: Disposable by Design

In a world of account fatigue and persistent tracking, **Kuikui** takes a
different approach. It is an anonymous, ephemeral collaboration tool designed
for the "here and now."

The core idea is simple: **Click, Share, Collaborate.**

- **No Accounts:** Identity is temporary.
- **No Database:** Data lives only as long as the session.
- **No Friction:** A URL is all you need.

Whether it's a quick code interview, a brainstorming session, or just showing a
friend a snippet of text, Kuikui provides a clean slate that vanishes when
you're done.

<!-- more -->

## Architecture & Developer Experience (DX)

Building a real-time app is complex, but setting it up shouldn't be. We
prioritized a seamless Developer Experience (DX) using a **Monorepo** structure
with robust tooling.

### 1. Monorepo Strategy

We use **npm workspaces** to keep the `backend` and `frontend` in sync while
maintaining separation of concerns. This allows us to share TypeScript
definitions directly, ensuring that the frontend and backend always speak the
same language.

### 2. Automated Workflows ("Rich Scripts")

We believe that `npm run dev` should just work. To achieve this, we built custom
tooling to handle the common annoyances of full-stack development.

**The "One Command" Experience:** When a developer runs `npm run dev`, a chain
of automated scripts executes:

1.  **Port Cleanup:** `scripts/port-cleanup.js` automatically detects and kills
    any zombie processes occupying ports 3001 or 5173. No more `EADDRINUSE`
    errors!
2.  **Environment Setup:** `scripts/env-setup.js` ensures `.env` files exist and
    are populated with safe defaults.
3.  **Validation:** Checks that all required variables are present.
4.  **Concurrency:** Finally, it launches both servers in parallel.

```json
// package.json
"scripts": {
  "dev": "npm run port:cleanup && npm run env:setup && npm run env:validate && concurrently ..."
}
```

## Code Snippets

### Type Safety Across Boundaries

One of the biggest sources of bugs in full-stack apps is the API contract
drifting between client and server. By sharing our `types` package, we enforce
strict contracts.

If the backend changes the `JoinRoomRequest` interface, the frontend build will
immediately fail, catching bugs at compile time rather than runtime.

```typescript
// shared/types/index.ts
export interface JoinRoomRequest {
  roomId: string;
  nickname: string;
  userId?: string; // Optional: for user persistence across sessions
}
```

### Real-time Engine: Taming the WebSocket

The heart of Kuikui is **Y.js** (CRDT) over **Socket.IO**. A naive
implementation would send a socket event for every single keystroke, which
scales poorly.

We implemented a **Smart Batching Layer** in our `SocketProvider`. It buffers
high-frequency updates (like cursor movements) and coalesces them into single
payloads, significantly reducing network overhead without compromising the
"real-time" feel.

```typescript
// frontend/src/services/socketProvider.ts
// Intelligent batching of document updates
this.doc.on("update", (update: Uint8Array) => {
  if (!this.isConnected) return;

  this.pendingDocUpdates.push(update);
  // Debounce: Wait 250ms to gather more keystrokes before sending
  this.docFlushTimer ??= setTimeout(() => this.flushDocUpdates(), 250);
});
```

## Gallery

### Initial Home Page

![Initial Home Page](https://github.com/thousandmiles/kuikui/raw/main/docs/home%20page.png)

### Create a Room

![Create a Room](https://github.com/thousandmiles/kuikui/raw/main/docs/join%20page.png)

### Send & Receive Message

![Send Message](https://github.com/thousandmiles/kuikui/raw/main/docs/send%20msg.png)
![Receive Message](https://github.com/thousandmiles/kuikui/raw/main/docs/receive%20msg.png)

### Editor Mode

![Editor Mode](https://github.com/thousandmiles/kuikui/raw/main/docs/edit%20mode.png)

### Collaboration

![Collaboration](https://github.com/thousandmiles/kuikui/raw/main/docs/collaboration.png)

## Refactoring Roadmap: Scaling Up

While the current implementation is perfect for a "hackathon" or small team
usage, we have a clear vision for scaling Kuikui to enterprise levels:

1.  **Persistence Layer (Redis):** Currently, room state is in-memory. We plan
    to introduce Redis to persist room state, allowing sessions to survive
    server restarts.
2.  **Horizontal Scaling:** By using the **Socket.IO Redis Adapter**, we can
    distribute rooms across multiple backend nodes, allowing us to support
    thousands of concurrent users.
3.  **E2E Testing:** We have solid unit test coverage (92% backend), but
    real-time apps need real-browser testing. We are planning to integrate
    **Playwright** to simulate multi-user collaboration scenarios automatically.

## Results

Kuikui proves that you don't need a heavy tech stack to build powerful
collaboration tools. By focusing on the core user experience and leveraging the
right libraries (Y.js, Socket.IO), we built a tool that feels magic to use.

We maintain high test coverage to ensure reliability:

### Backend Coverage

| Type       | Percentage |
| ---------- | ---------- |
| Statements | 92.93%     |
| Branches   | 85.62%     |
| Functions  | 91.11%     |
| Lines      | 92.93%     |

### Frontend Coverage

| Type       | Percentage |
| ---------- | ---------- |
| Statements | 76.58%     |
| Branches   | 83.79%     |
| Functions  | 71.05%     |
| Lines      | 76.58%     |

## Links

- **Source Code:** [GitHub Repository](https://github.com/thousandmiles/kuikui)
