---
title: "The Silent Squeezer: How HPACK Compresses the Web"
date: "2026-06-10T09:00:00.000Z"
description: ""HTTP/1.1 wasted massive amounts of bandwidth by sending the same headers over and over. HTTP/2 fixed this with HPACK—a stateful, secure compression engine that can shrink your headers by 95%.""
---

If you’ve ever looked at a raw HTTP/1.1 request, you’ve seen the waste. Every single time you click a link, your browser sends the exact same `User-Agent`, the same `Accept` headers, and the same massive `Cookie`. For a tiny 100-byte JSON response, you might be sending 2KB of redundant headers. On a slow mobile network, this is a performance killer.

**HPACK** (specified in RFC 7541) is the specialized compression algorithm that fixed this in HTTP/2. It is more than just a minifier; it is a sophisticated, stateful system that uses synchronized tables and custom bit-coding to ensure that no redundant information ever crosses the wire.

### 1. The Three Pillars of HPACK

HPACK uses three distinct mechanisms that work together to squeeze every unnecessary bit out of the connection.

#### A. The Static Table: The Built-in Dictionary
The first thing HPACK does is define a **Static Table** of 61 common header fields. 
- **Index 2:** `:method: GET`
- **Index 8:** `:status: 200`
- **Index 58:** `user-agent`
If you’re sending a standard GET request, HPACK doesn't send the string "GET." It sends the number `2`. This turns a 12-byte string into a 1-byte integer.

#### B. The Dynamic Table: Learning Your Habits
The Static Table only covers the basics. For everything else (like your specific session cookie), HPACK uses a **Dynamic Table**. 
Both the client and the server maintain a synchronized copy of this table. When you send a custom header for the first time, HPACK sends the full string, but then it adds it to the Dynamic Table. On the next request, it just sends the index.
- **Eviction:** The Dynamic Table has a size limit (usually 4KB). When it gets full, it uses a **FIFO (First-In, First-Out)** algorithm to evict old entries and make room for new ones.

#### C. Static Huffman Coding: Squeezing the Strings
If a header isn't in either table, HPACK doesn't send it as raw ASCII. It uses a **Static Huffman Table** specifically tuned for HTTP headers. 
Characters that appear frequently in URLs and headers (like `/`, `.`, and lowercase letters) are given very short bit-sequences (as few as 5 bits), while rare characters are longer. On average, this saves about 30% on the size of literal strings.

### 2. The CRIME Attack: Why HPACK isn't GZIP

You might wonder: why didn't we just use standard GZIP for headers? The answer is a security vulnerability called **CRIME** (Compression Ratio Info-leak Made Easy).

In GZIP, the compression ratio depends on how much the data repeats. An attacker could inject a string into your request and see if the packet size got smaller. If it did, it meant the injected string matched part of your secret cookie. By doing this repeatedly, they could guess your cookie character-by-character.

HPACK prevents this by being **non-speculative**:
1.  It only matches **full** header names or values, not partial substrings.
2.  It uses a **Static** Huffman table that an attacker cannot influence.
3.  It allows sensitive headers (like `Cookie` or `Authorization`) to be marked as **"Never Indexed,"** ensuring they are never stored in the Dynamic Table where they could be probed.

### 3. The Address Space

HPACK uses a single, continuous address space for its indexes.
- **1 to 61:** Map to the Static Table.
- **62 and above:** Map to the Dynamic Table.

When a decoder sees a byte with the high bit set (an "Indexed Header Field"), it knows the remaining bits are an index into this address space. It looks up the entry and immediately knows both the header name and the value.

```text
ASCII HPACK Decoding:
[ High Bit: 1 ] [ Bits 0-6: 0000010 ]
       |                |
   "Indexed"      Index 2 (Static Table)
       |                |
       v                v
   Result: ":method: GET" (Extremely Smashed!)
```

### Conclusion

HPACK is a masterpiece of protocol optimization. It recognizes that in a world of high-latency mobile networks, every byte counts. By moving from the "stateless waste" of HTTP/1.1 to the "stateful efficiency" of HPACK, we’ve effectively removed the header overhead from the critical path of web performance. It is a reminder that the best way to move data faster is to realize that you’ve already sent most of it before.
