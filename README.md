# System Design: Beyond the Diagram

<img width="1672" height="941" alt="Why_ Start With Requirements" src="https://github.com/user-attachments/assets/32f6aec4-75d3-4053-8486-65f32f9656e4" />

**One system. One challenge. One engineering lesson.**

A growing collection of my **system design interview practice notes** — exploring how everyday products work at scale, what can go wrong, and the trade-offs engineers make to keep them reliable.

> **What happens if a payment succeeds, but the server crashes before receiving the response?**
>
> That's the kind of question I'm using to go beyond memorizing architecture diagrams and think about failure handling, data consistency, scalability, and real-world constraints.

**11 system design topics** · **Interview preparation** · **Architecture & trade-offs** · **Learning in public**

---

## Explore the systems

Each topic starts with a practical problem and leads into architectural decisions worth discussing in an interview.

| # | System | The question worth asking |
|:--:|---|---|
| 01 | [Payment System](./Payment%20System/) | What happens when a payment goes through but the response is lost? |
| 02 | [Rate Limiter](./Rate%20Limiter/) | How do we protect an API from sudden bursts of requests? |
| 03 | [Chat System](./Chat%20System/) | How do messages reach users who are offline? |
| 04 | [News Feed](./News%20Feed/) | How can a feed stay responsive as millions of people post? |
| 05 | [Ride-Hailing Platform](./Ride%20Hailing%20Platform/) | How do we match riders with nearby drivers quickly? |
| 06 | [Ticket Purchasing System](./Ticket%20Purchasing%20System/) | How do we prevent two people from booking the same seat? |
| 07 | [URL Shortener](./URL%20Shortener%20%28Bitly%29/) | How can short links be generated uniquely at scale? |
| 08 | [Key-Value Store](./Key%20Store%20Value%20System/) | What happens when a node in a distributed store fails? |
| 09 | [Notification System](./Notification%20System/) | How do we deliver large volumes of notifications reliably? |
| 10 | [File Storage (Drive / Dropbox)](./File%20Storage%28Drive%2CDropbox%29/) | How do we handle large file uploads and interrupted transfers? |
| 11 | [Unique ID Generator](./Unique%20ID%20Generator/) | How can distributed services generate IDs without collisions? |

### Cross-topic review

- [System Design Cheat Sheet](./Cheat%20set.md) — a quick reference for revising common concepts.

---

## What I'm practicing

- **Clarifying requirements:** what the system needs to do, and what matters most.
- **Estimating scale:** traffic, storage, latency, and expected growth.
- **Choosing building blocks:** databases, caches, queues, load balancers, and services.
- **Handling failures:** retries, duplicate requests, downtime, and recovery.
- **Explaining trade-offs:** why a design fits certain requirements — and where it may struggle.

## How to use these notes

1. **Pick a scenario** from the table above and try to design it yourself first.
2. **Compare approaches** using the notes in that topic folder.
3. **Challenge the design:** What breaks under heavy traffic? What if a service fails? What could be simpler?
4. **Explain it aloud** as though you're in a system design interview.

Some folders include both **AI-assisted study notes** and **my own manual notes**. I'm using these as learning material, not claiming the architectures are production implementations or universally correct answers.

## Why I'm sharing this

I'm preparing for software engineering interviews and using this repository to make my learning more structured. My biggest takeaway so far is that **system design isn't about drawing the perfect diagram — it's about making informed decisions under constraints**.

I plan to share short lessons from these topics on LinkedIn as **System Design: Beyond the Diagram**, with a focus on one engineering problem at a time.

If you're studying system design too, feel free to explore the notes, discuss an alternative approach, or point out a trade-off I missed.

---

*This is a continuously evolving learning repository. The examples are for study and interview practice, not production-ready reference architectures.*
