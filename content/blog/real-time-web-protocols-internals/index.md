---
title: "The Real-Time Dilemma: WebSockets vs. SSE vs. HTTP/2 Push"
date: "2026-06-10T09:00:00.000Z"
description: ""Choosing a real-time protocol is no longer just about speed. It’s a trade-off between the bi-directional power of WebSockets, the simplicity of SSE, and the hard lessons learned from the failure of HTTP/2 Push.""
---

If you’ve ever built a chat app, a live dashboard, or a collaborative editor, you’ve faced the same technical hurdle: how do you get the server to talk to the client without waiting for a request? For twenty years, we relied on "Long Polling" (a hack where the client just waits for the server to eventually answer). But today, we have three distinct architectures for real-time data, and each one makes a different bet on the future of the web.

### 1. WebSockets: The Full-Duplex Phone Call

WebSockets are the "nuclear option" of real-time communication. They provide a persistent, full-duplex TCP connection between the client and the server.

-   **The Handshake:** It starts with a standard HTTP request containing an `Upgrade: websocket` header. If the server agrees, it responds with a `101 Switching Protocols` status. At that moment, the HTTP layer is discarded, and the connection becomes a raw, stateful TCP pipe.
-   **The Internals:** WebSockets use a very lightweight framing protocol (RFC 6455). A message can have as little as 2 bytes of overhead. This makes it incredibly efficient for high-frequency updates, like a multiplayer game where you’re sending player coordinates 60 times a second.
-   **The Catch:** Because it’s a stateful connection, horizontal scaling is difficult. If you have ten server nodes, User A might be connected to Node 1, while User B is on Node 10. To send a message between them, you need a backend "backplane" like **Redis Pub/Sub** to synchronize state across your cluster.

### 2. Server-Sent Events (SSE): The Radio Broadcast

**Server-Sent Events** are the "underrated" alternative. While WebSockets are bidirectional, SSE is unidirectional: it only allows the server to push data to the client.

-   **The Mechanism:** SSE is just a standard HTTP request with a special `Content-Type: text/event-stream`. The server simply never closes the connection. It sends data in a simple text format: `data: Hello world\n\n`.
-   **The Superpower: Auto-Reconnection.** Unlike WebSockets, where you have to write complex logic to handle a dropped connection, browsers have built-in support for SSE. If the connection drops, the browser automatically tries to reconnect, sending a `Last-Event-ID` header so the server knows exactly where to resume the stream.
-   **Modern Scaling:** Under HTTP/1.1, SSE was limited to 6 connections per domain. But under **HTTP/2**, SSE is multiplexed into a single TCP connection, effectively removing this limit and making it as scalable as any other web request.

### 3. HTTP/2 Server Push: The Noble Failure

Five years ago, we were told that **HTTP/2 Server Push** would revolutionize the web. The idea was simple: when a client asks for `index.html`, the server knows it will also need `style.css`, so it "pushes" the CSS file into the browser’s cache before the browser even asks for it.

**Why it died:** 
In 2024, Chrome and Firefox officially deprecated Server Push. It failed for three technical reasons:
1.  **Cache Awareness:** The server often pushed files the client already had, wasting precious mobile bandwidth.
2.  **HOL Blocking:** Pushing large files could accidentally block the more important HTML or JavaScript data.
3.  **Complexity:** It was almost impossible for developers to tune correctly.

Today, the web has moved to **103 Early Hints**, where the server sends a "hint" to the browser to start fetching resources while the server is still busy generating the main page. It’s "Pull, don't Push," and it works much better.

```text
ASCII Real-Time Architecture:
[ WebSockets ] --(TCP Handshake)--> [ Stateful Bi-directional Pipe ]
[ SSE        ] --(HTTP Stream)----> [ Stateless Server-to-Client Stream ]
[ Push/Hints ] --(H2 Promise)-----> [ Pre-emptive Cache Warming ]
```

### Summary: When to use what?

| Protocol | Best For... | Why? |
| :--- | :--- | :--- |
| **WebSockets** | Gaming, Chat, Collaborative Tools | Bi-directional, binary support, lowest latency. |
| **SSE** | Stock Tickers, Token Streaming (AI) | Simple HTTP, auto-reconnection, easy scaling. |
| **103 Hints** | Asset Loading (JS/CSS) | Browser-controlled, avoids cache waste. |

### Conclusion

The "real-time" web is maturing. We’ve moved past the "WebSockets for everything" phase. Today, a senior architect uses WebSockets for the complex bidirectional stuff, SSE for simple data streams (like ChatGPT’s typing effect), and trusts standard browser preloading for assets. It’s a reminder that in software engineering, the best tool is rarely the one with the most features; it’s the one that solves your problem with the least amount of hidden complexity.
