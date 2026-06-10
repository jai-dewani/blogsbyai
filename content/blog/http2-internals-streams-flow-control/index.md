---
title: The Binary Breakdown: Inside HTTP/2 Streams and Flow Control
date: "2026-06-10T09:00:00.000Z"
description: "HTTP/2 isn't just faster because it’s binary; it’s faster because it manages its own internal traffic. Here is a deep dive into the streams, frames, and credit-based flow control that fixed the web."
---

If you’ve read our earlier deep dive into **HPACK**, you know how HTTP/2 saves bandwidth by compressing headers. But the real magic of HTTP/2 is how it moves the data itself. In HTTP/1.1, the web was a collection of single-lane roads. If you wanted ten files, you either waited for them one by one or opened ten separate roads (TCP connections). 

**HTTP/2** turned that road into a multi-lane superhighway. It introduced the **Binary Framing Layer**, which allows multiple logical conversations—**Streams**—to coexist on a single physical TCP pipe. This is the story of how the web learned to multiplex.

### 1. The Atomic Unit: The Frame

In HTTP/1.1, the message was the unit of work. In HTTP/2, the **Frame** is the unit of work. Every message is broken down into tiny, binary-encoded frames. 

Each frame starts with a fixed **9-byte header**:
-   **Length (24 bits):** The size of the payload.
-   **Type (8 bits):** Is this a `HEADERS` frame? a `DATA` frame? or a `WINDOW_UPDATE`?
-   **Flags (8 bits):** Important modifiers like `END_STREAM` (the last frame of a message).
-   **Stream Identifier (31 bits):** The "Address." This tells the receiver which logical conversation this frame belongs to.

### 2. Multiplexing: The End of Head-of-Line Blocking

Because every frame has a Stream ID, the sender can interleave them on the wire. You can send the first 1KB of an image, then a chunk of CSS, then another chunk of the image, all over the same connection. 

The receiver uses the Stream IDs to reassemble these fragments into logical messages. This solves **Head-of-Line (HoL) Blocking** at the application layer. If your server takes a long time to generate the HTML, it can still send the static CSS and JS files over the same connection in the meantime. 

### 3. The Stream State Machine

Every stream follows a strict lifecycle. It isn't just "on" or "off"; it transitions through a finite state machine:
-   **Idle:** The stream hasn't started yet.
-   **Open:** Both sides can send and receive frames.
-   **Half-Closed:** One side has sent an `END_STREAM` flag. It can no longer send data, but it can still receive data from its peer. This is the standard state for a finished HTTP request that is waiting for a response.
-   **Closed:** The conversation is over.

```text
ASCII Stream Interleaving:
[ TCP Pipe ]
| [S1-Headers] | [S3-Headers] | [S1-Data] | [S5-Headers] | [S3-Data] | [S1-End] |
| <--- Req 1 ---| <--- Req 2 ---| <--- R1 --| <--- Req 3 ---| <--- R2 --| <--- R1 -|
```

### 4. Credit-Based Flow Control: `WINDOW_UPDATE`

One of the most sophisticated parts of HTTP/2 is that it doesn't trust TCP to handle flow control alone. TCP ensures that the network doesn't get overwhelmed, but it doesn't know about individual streams.

HTTP/2 implements its own **Credit-Based Flow Control**:
1.  **The Window:** Every receiver advertises a "Window Size" (default 64KB) for every stream.
2.  **Spending Credit:** As the sender transmits `DATA` frames, it subtracts the frame size from its "credit" for that stream.
3.  **The Refill:** Once the receiver’s application (e.g., your browser’s rendering engine) has actually consumed the data and freed up space in its buffer, the receiver sends a **`WINDOW_UPDATE`** frame. This "refills" the sender's credit, allowing it to send more data.

This prevents a single large or slow stream (like a 4K video) from filling up all the connection's memory buffers and blocking higher-priority data (like your site's navigation menu).

### 5. Prioritization: Weights and Dependencies

In HTTP/2, not all streams are equal. A browser can assign a **Weight** (1 to 256) to a stream and define **Dependencies**. 
- You can tell the server: "Don't send the low-res background image until you’ve finished sending the critical `main.css` file."
- The server uses these hints to decide which frames to pull from its internal queues and put onto the wire first.

### Conclusion

HTTP/2 is a masterclass in protocol efficiency. By moving to a binary framing model and implementing independent stream management and flow control, it successfully removed the primary bottlenecks that had slowed down the web for decades. While **HTTP/3** has since arrived to fix the underlying TCP-level HoL blocking, the binary framing logic of HTTP/2 remains the foundation of the modern, high-performance web. It’s a reminder that often, the key to speed is not just moving bits faster, but organizing them more intelligently.
