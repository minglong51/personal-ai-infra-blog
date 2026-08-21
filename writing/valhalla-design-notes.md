---
title: "Valhalla: an attention layer for a fleet of agents"
date: 2026-07-24
readingMinutes: 6
summary: The operations layer for my agent fleet — agents post only what needs a decision to one attention feed, over a shared knowledge memory. Design notes on exception-driven surfaces and the four friction laws.
syndicate: [substack, x]
canonical: https://www.minglongpan.com/writing/valhalla-design-notes
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/valhalla-design-notes). That version is canonical and may be updated; this file is a point-in-time mirror.

Valhalla is the piece of my personal infrastructure everything else hangs off: the operations layer between a fleet of agents and one human's attention. The premise is that my attention is the scarcest resource in the system — scarcer than compute, scarcer than tokens — so the architecture optimizes for it explicitly. This note covers the design. Topology, hosts, and credentials stay private.

## The problem: agents produce noise by default

Give five agents standing jobs — monitors, digests, watches — and they will generate a river of state updates that reads like productivity. It isn't. Every notification that doesn't require a decision trains the human to ignore the channel, and a trained-out channel is worse than no channel. The first design rule of Valhalla followed directly: **agents publish state, not notifications.** A monitor's output lands in a shared store; the *surface* decides whether any of it deserves to interrupt me.

## One attention feed, exception-driven

Valhalla's front door is a single feed where every entry must pass a test: **does this need a decision now?** A card exists only when it's actionable — overdue, due today, a tripwire that just crossed. Healthy state is a quiet line or nothing at all. Quiet days produce no feed; the system being calm is not news.

Two mechanics keep the feed honest:

- **One loud action per surface.** Each view gets a single primary action; everything else renders quieter. The moment two things shout, the ranking has failed and I stop trusting it.
- **Completion collapses.** A done item shrinks to a dim one-liner with undo and hands its visual weight to the next item. A checked box that leaves the card full-size reads as "nothing happened," which erodes the feed's credibility one tiny lie at a time.

## The shared knowledge memory

Under the feed sits a memory layer every agent reads and writes — working notes with TTLs, durable build notes, and red lines (no internal topology, no private data, ever). This is what makes a fleet behave like one organism: the finance monitor, the site builder, and the drill scheduler don't talk to each other, but they all read the same ground truth. Most fleet failures I've had trace back to memory failures — an agent acting on a stale note — so the memory layer gets the same engineering care as the agents themselves: ownership, expiry, and a rule that nothing duplicates what code or git already records.

## The four friction laws

Valhalla's apps share four laws, learned the hard way:

1. **Capture goes to the conversation.** You never open an app to log something; you message the bot. Logging that requires opening an app is logging that won't happen.
2. **The ask travels to where I already look.** Apps publish one-line state into the feed via their own read APIs — never a new destination to check.
3. **Starting is logging.** A start button runs a timer whose finish *is* the log entry. No post-hoc bookkeeping anywhere.
4. **Deep links land on the verb, not the lobby.** A link opens mid-task, never a home page to navigate.

Each law exists because its violation was observed in the wild and killed a habit. They're cheap to state and expensive to hold — every new surface gets audited against all four before it ships.

## What it isn't

Valhalla is not a dashboard empire. The feed is intentionally sparse; most days it shows one thing or nothing. The ambition is the opposite of engagement: a system I check for thirty seconds because it only ever shows me what changed. If a feature would make me spend more time in it, that feature is a bug with good marketing.

---

*This note is the "Valhalla" entry in the index of this site. It's private by design — the philosophy is the shareable part.*
