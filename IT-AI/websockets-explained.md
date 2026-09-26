# WebSockets — Explained Simply

## What is a WebSocket?

A **WebSocket** is a persistent, two-way connection between a client
(browser, app, script) and a server. Once opened, either side can send
messages to the other **at any time**, with no need to re-ask ("poll")
for updates.

Compare it to HTTP/cURL:
- **HTTP/cURL** = sending a letter and waiting for one reply, then the
  conversation ends. Want new info? Send another letter.
- **WebSocket** = a phone call. You pick up once, and then both people can
  talk whenever they want, back and forth, until someone hangs up.

This makes WebSockets the standard choice for anything "live": chat apps,
stock tickers, multiplayer games, live dashboards, notifications.

---

## How it works

1. **Handshake (upgrade):** the client sends a normal HTTP request, but
   with special headers asking to "upgrade" the connection:
   ```
   GET /chat HTTP/1.1
   Host: example.com
   Upgrade: websocket
   Connection: Upgrade
   ```
2. **Server agrees:** if it supports WebSockets, it replies `101 Switching
   Protocols`, and from this point the plain HTTP connection turns into a
   WebSocket connection.
3. **Open connection:** the underlying TCP connection stays open. No more
   HTTP requests/responses — now it's just raw messages flowing both ways.
4. **Messages, either direction, anytime:**
   - Server can push data to client without being asked (e.g. "new chat
     message arrived").
   - Client can send data to server anytime (e.g. "user typed a message").
5. **Close:** either side sends a close frame, or the connection drops
   (network issue, timeout, etc.).

### URL scheme
WebSocket URLs use `ws://` (plain) or `wss://` (secure, like `https`):
```
ws://example.com/socket
wss://example.com/socket   ← secure, use this in production
```

---

## Real use-case examples

- **Chat apps** (Slack, Discord, WhatsApp Web) — messages appear instantly
  without refreshing.
- **Live price feeds** (crypto exchanges, stock tickers) — prices update
  every second without you clicking "refresh."
- **Multiplayer games** — player positions sync in real time.
- **Live dashboards** — server pushes new metrics as they happen.
- **Collaborative editing** (Google Docs-style) — see other people's
  cursor and edits live.
- **Notifications** — "someone liked your post" pops up without a page
  reload.

---

## Running a WebSocket client in Python

### Using the `websockets` library (most common)
Install: `pip install websockets`

```python
import asyncio
import websockets

async def main():
    uri = "wss://example.com/socket"
    async with websockets.connect(uri) as ws:
        # send a message
        await ws.send("hello server")

        # receive messages in a loop
        async for message in ws:
            print("received:", message)

asyncio.run(main())
```

Send and receive one message each:
```python
import asyncio
import websockets

async def main():
    async with websockets.connect("wss://echo.websocket.org") as ws:
        await ws.send("ping")
        reply = await ws.recv()
        print(reply)  # "ping"

asyncio.run(main())
```

---

## Running a WebSocket client in JavaScript

### In the browser or Node.js (built-in `WebSocket` API)
```javascript
const ws = new WebSocket("wss://example.com/socket");

// connection opened
ws.onopen = () => {
  console.log("connected");
  ws.send("hello server");
};

// message received from server
ws.onmessage = (event) => {
  console.log("received:", event.data);
};

// connection closed
ws.onclose = () => {
  console.log("disconnected");
};

// error
ws.onerror = (err) => {
  console.error("error:", err);
};
```

*(Node.js has `WebSocket` built in since v22; for older Node versions,
install the `ws` package: `npm install ws`.)*

### Using the `ws` package in Node.js (server or client)
```javascript
const WebSocket = require("ws");

const ws = new WebSocket("wss://example.com/socket");

ws.on("open", () => ws.send("hello server"));
ws.on("message", (data) => console.log("received:", data.toString()));
ws.on("close", () => console.log("disconnected"));
```

---

## A minimal WebSocket server (for context)

### Python (`websockets` library)
```python
import asyncio
import websockets

async def handler(websocket):
    async for message in websocket:
        print("client said:", message)
        await websocket.send(f"echo: {message}")

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        await asyncio.Future()  # run forever

asyncio.run(main())
```

### JavaScript (`ws` package)
```javascript
const { WebSocketServer } = require("ws");
const wss = new WebSocketServer({ port: 8765 });

wss.on("connection", (ws) => {
  ws.on("message", (data) => {
    console.log("client said:", data.toString());
    ws.send(`echo: ${data}`);
  });
});
```

---

## WebSocket vs. HTTP polling vs. Server-Sent Events (SSE)

| Approach | Direction | How it works | Best for |
|---|---|---|---|
| **HTTP polling** | Client → Server (repeated) | Client asks "anything new?" every N seconds | Simple, infrequent updates |
| **Server-Sent Events (SSE)** | Server → Client only | Server streams updates over one HTTP connection | One-way live feeds (notifications, logs) |
| **WebSocket** | Both ways, anytime | Persistent connection, either side sends anytime | Chat, games, anything needing two-way real-time |

---

## Quick equivalence cheat sheet

| Concept | Python (`websockets`) | JavaScript (browser/Node) |
|---|---|---|
| Connect | `websockets.connect(uri)` | `new WebSocket(url)` |
| Send | `await ws.send(msg)` | `ws.send(msg)` |
| Receive (one) | `await ws.recv()` | `ws.onmessage = (e) => ...` |
| Receive (loop) | `async for msg in ws:` | `ws.onmessage = (e) => ...` (fires per message) |
| Close | `await ws.close()` | `ws.close()` |

---

## Why this matters

Whenever you see "real-time," "live updates," or "push notifications" in a
product spec, WebSockets (or SSE, for one-way cases) are usually the right
tool — regular HTTP/cURL-style requests can't have the server proactively
send data without the client asking first.
