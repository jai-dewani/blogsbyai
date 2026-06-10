---
title: "The Internet’s Trust Crisis: Inside BGP Hijacking"
date: "2026-06-10T09:00:00.000Z"
description: ""The Border Gateway Protocol (BGP) was built on a handshake and a promise. Today, that trust is a global security liability that allows single actors to redirect the entire world's traffic.""
---

If you’ve read our earlier deep dive into **BGP Internals**, you know that BGP is the glue of the internet. It is a protocol that manages the complex, multi-billion dollar connections between thousands of independent networks. But there is a dark side to BGP: it is fundamentally built on trust. By default, a BGP router believes any advertisement it receives from a neighbor. 

This inherent trust is what makes **BGP Hijacking** one of the most significant and persistent threats to the stability of the global internet.

### 1. The Anatomy of a Hijack: Lying to the World

BGP hijacking happens when an Autonomous System (AS) announces an IP prefix that it doesn't actually own. 

There are two primary ways to do this:
-   **Prefix Hijacking:** An attacker announces your IP prefix as their own. Some routers will see this new path as "better" (shorter) and start sending your traffic to the attacker.
-   **Sub-prefix Hijacking:** This is the more effective version. The attacker announces a more specific portion of your network (e.g., a `/24` instead of your `/23`). Because BGP always prefers the **most specific route**, traffic from all over the world will instantly divert to the attacker, bypassing your legitimate network entirely.

### 2. RPKI: The Cryptographic Gatekeeper

To combat this, the internet has deployed **RPKI (Resource Public Key Infrastructure)**. RPKI uses a distributed database of **ROAs (Route Origin Authorizations)** to cryptographically link a specific IP prefix to its legitimate owner.

When a router receives a BGP update, it checks the **Origin AS** against the RPKI database. 
- If they match, the route is **Valid**.
- If the AS isn't authorized for that prefix, the route is **Invalid** and the router drops it. This is called **Route Origin Validation (ROV)**.

### 3. AS-PATH Spoofing: The RPKI Bypass

While RPKI has made simple hijacks harder, it has also forced attackers to become more sophisticated. The new frontier of BGP attacks is **AS-PATH Spoofing**.

RPKI only validates the **last step** of the journey (the Origin AS). It doesn't look at the path taken to get there. In an AS-PATH spoofing attack, an attacker (AS 666) announces a route to your network (AS 100) but includes your legitimate AS in the path: `[666, 100]`. 

To RPKI, this looks like a valid route from your network that just happened to go through AS 666. It passes the validation check, allowing the attacker to redirect your traffic while still wearing a badge of cryptographic legitimacy.

```text
ASCII AS-PATH Spoofing:
[ Legitimate: AS 100 ] <--- (Trust) --- [ Peer ]
           |
[ Attacker: AS 666 ] --(Lies)--> [ Peer ]
     "I have a path to 100: [666, 100]"
           |
[ RPKI Check ] --(Looks at 100)--> "VALID!"
```

### 4. The Path to Security: ASPA and BGPsec

The community is currently working on two major fixes for this "trust gap":

-   **BGPsec:** This adds a digital signature to every single hop in the BGP path. While technically perfect, it has failed to gain adoption because it is incredibly CPU-intensive and requires every network in the world to upgrade simultaneously.
-   **ASPA (AS Path Authorization):** A more pragmatic approach. It allows an AS to publish a list of its authorized **Upstream Providers**. By checking this list, a router can detect if a path includes an unauthorized or suspicious jump, catching most spoofs and accidental route leaks.

### Conclusion

BGP Hijacking is a reminder that the internet was not designed for the modern world. It is a 1980s protocol running a 2020s global economy. While RPKI and ASPA are making the network safer, the fundamental vulnerability remains: the internet is only as strong as its weakest peering relationship. It is a masterclass in the challenge of securing a system that no one truly owns and everyone relies on.
