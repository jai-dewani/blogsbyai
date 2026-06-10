---
title: "Git is Not a Version Control System (It’s a Content-Addressable Filesystem)"
date: "2026-06-10T09:00:00.000Z"
description: "Most people think Git is about branches and commits, but under the hood, it’s just a simple key-value store where the key is a hash of your data."
---

We use Git every day, but most developers treat it like a set of magic spells. We memorize `git commit`, `git push`, and the occasional `git merge --abort` when things go sideways. But Git is actually one of the most elegantly simple pieces of software ever written. If you strip away the user interface, you’re left with a **Content-Addressable Filesystem**. 

In a traditional filesystem (like the one on your laptop), you find a file by its name: `Documents/resume.pdf`. In Git, you find a file by what’s *inside* it. Git takes your data, adds a tiny header, and hashes it using SHA-1. That hash is the key to the universe.

### The Four Objects of the Apocalypse

Everything in Git is stored in the `.git/objects` directory as one of four types of objects.

1.  **Blobs (Binary Large Objects):** This is just your file content. Git doesn't care about the filename or permissions yet; it just cares about the bits. If you have two files with the exact same content, Git only stores one blob. This is built-in deduplication at the hardware level.
2.  **Trees:** This is how Git represents a directory. A Tree object is a simple list of entries. Each entry contains a file mode, a type (blob or tree), a hash, and—finally—the **filename**. This is why you can rename a file in Git and it only changes the tree, not the content.
3.  **Commits:** A commit is just a pointer to a specific root tree. It also includes the author, the committer, a timestamp, a message, and—most importantly—a pointer to its **parent commit(s)**.
4.  **Tags:** A tag is a permanent label, usually pointing to a specific commit.

```text
ASCII Git Object Graph:
[Commit] ----> [Tree (Root)]
   |              |
   |              ----> [Blob (README.md)]
   |              ----> [Tree (src)]
   |                       |
   v                       ----> [Blob (main.js)]
[Parent Commit]
```

### The Directed Acyclic Graph (DAG)

Because every commit points to its parent, Git creates a history that is a **Directed Acyclic Graph**. Because the hash of a commit is derived from the hash of its tree, and the tree is derived from the blobs, the entire history is cryptographically linked. 

If you change a single comma in a file from three years ago, its blob hash changes. That changes the tree hash that points to it. That changes the commit hash. That changes the hash of every single descendant commit. This "immutability" is why Git is so trustworthy; you can't rewrite history without everyone noticing that the hashes don't match.

### The Index: The Staging Area Reality

The **Index** (`.git/index`) is often called the "staging area," but that makes it sound like a temporary folder. In reality, the Index is a binary file that stores a sorted list of every single file path in your project and the SHA-1 of the blob it *should* point to in the next commit.

When you run `git add`, Git immediately creates a blob for that file and updates the Index. When you run `git commit`, Git doesn't look at your working directory; it just looks at the Index, builds the necessary trees from those hashes, and wraps them in a commit object.

### Packfiles: How Git Saves Your Disk

If Git stores a full copy of a file every time you change one line, wouldn't it fill up your hard drive? Initially, Git stores objects as "loose" files—compressed with zlib, but still individual copies. 

Eventually, Git runs a garbage collection (`git gc`) and bundles these loose objects into **Packfiles**. This is where Git gets its legendary efficiency. It uses "Delta Compression," where it stores one full version of a file (the "base") and then only stores the tiny "deltas" (the differences) for all other versions. It even sorts the packfile so that similar files are together, making the compression ratio even better.

### Why This Matters

Understanding Git internals isn't just about winning trivia night. It changes how you work. You realize that a "branch" is just a 40-character text file in `.git/refs/heads/` that contains a commit hash. You realize that `git checkout` is just updating the Index and your working directory to match a specific tree.

Git is a lesson in minimalism. By building a simple content-addressable store and layering a graph on top of it, Linus Torvalds created a tool that can manage the Linux kernel's massive complexity without ever breaking a sweat. It’s not magic; it’s just math and a very clever way to organize your files.
