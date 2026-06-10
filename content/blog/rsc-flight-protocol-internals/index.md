---
title: "The Flight Protocol: Inside the RSC Wire Format"
date: "2026-06-10T09:00:00.000Z"
description: "React Server Components (RSC) are not just about server-side rendering; they are a sophisticated data-streaming architecture that uses a specialized, line-based format to move the virtual DOM down the wire."
---

If you’ve opened the Network tab while using a modern Next.js or React app, you’ve probably seen a weird response that looks like a mix of JSON and gibberish. It’s not HTML, and it’s not standard JSON. This is the **Flight Protocol**—the internal wire format for **React Server Components (RSC)**.

Most developers understand the high-level benefit of RSC: "zero-bundle-size components." But the real magic happens in how React serializes your component tree and streams it to the browser. It is a masterpiece of compact, incremental data transfer.

### 1. The Stream: Line-by-Line Execution

Unlike standard JSON, which must be fully downloaded before it can be parsed, the RSC format is **line-based**. Each line is a self-contained "chunk" prefixed with a type identifier. This allows the browser's stream reader to process and render parts of the page as soon as they arrive, even if a heavy database query is still running on the server.

### 2. The Token Language: J, M, and S

The Flight protocol uses single-character tokens to define what each line represents:

-   **`J` (Virtual DOM):** This is a serialized React element. It looks like standard JSON but uses a special compact syntax. For example, `["$","div",null,{"children":"Hello"}]` is the serialized version of `<div>Hello</div>`.
-   **`M` (Module Reference):** This is how the server tells the client about a **Client Component**. It doesn't send the code; it sends the ID of the JavaScript bundle that the browser needs to fetch.
-   **`S` (Symbol):** Used for internal React symbols, ensuring that things like `Symbol.for('react.element')` are preserved across the network boundary.
-   **`H` (Hint):** Tells the browser to start preloading assets like CSS or fonts while the data is still streaming.

### 3. The Reference System: `$L` and `$1`

The RSC format is designed for **deduplication**. If the same component or data appears multiple times in your tree, the server only sends it once. It then uses references like `$1` or `$L1` (for Lazy/Module) to point back to that data.

This reference system is also how **Suspense** works. If a component is still loading, the server sends a placeholder reference. When the data finally resolves, a new line is pushed to the stream with the actual data, and the client-side React runtime "plugs" it into the correct spot.

```text
ASCII Flight Payload:
M1:{"id":"./Counter.js","name":"Counter"}  <-- Client Component Ref
J0:["$","div",null,{"children":["$L1"]}]    <-- The Tree, referencing M1
```

### 4. Why not just send HTML?

This is the most common question about RSC. Why go through the trouble of this complex format when we have SSR (Server Side Rendering) that sends HTML?

The answer is **Reconciliation**. 
- HTML is static. Once it's in the DOM, it's hard to change without a full reload or complex JavaScript.
- The RSC format is a representation of the **Virtual DOM**. This allows React to "diff" the new server-rendered UI against the current client-side state. 

If you have a video playing or a focused text input, and you perform a "Server Action" that refreshes the page, the RSC stream allows React to update the data *without* interrupting your video or losing your cursor position. You get the speed of the server with the statefulness of the client.

### 5. Hydration: The Weight of the Static

In traditional SSR, every single element on the page—even a static footer—must be "hydrated" by JavaScript on the client. This is a massive waste of CPU time.

With RSC, the components that are rendered as `J` tokens (Server Components) **never hydrate**. The browser receives the UI and renders it, but there is zero JavaScript overhead for them. Only the components marked with `M` (Client Components) trigger hydration. This is how React finally solved the "Hydration Tax" that has plagued web performance for a decade.

### Conclusion

React Server Components represent the most significant architectural change to the web since the introduction of AJAX. By creating a specialized, streamable wire format that bridges the gap between server-side execution and client-side reconciliation, React has created a platform that is both incredibly fast and remarkably interactive. It’s a reminder that sometimes the best way to improve a system is to change the way the parts talk to each other.
