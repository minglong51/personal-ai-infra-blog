---
title: "Three agent frameworks converged on a control-plane protocol this month. I run my fleet on tmux and chose not to adopt it."
date: 2026-07-20
readingMinutes: 7
summary: "why I didn't adopt the converging protocol"
syndicate: []
canonical: https://www.minglongpan.com/writing/three-agent-frameworks-converged-on-a-control-plane-protocol
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/three-agent-frameworks-converged-on-a-control-plane-protocol). That version is canonical and may be updated; this file is a point-in-time mirror.

Three agent frameworks converged on the same idea this month: give a fleet of AI agents a structured, addressable control plane instead of an ad hoc one. I read all three and decided to keep running mine on a shell one-liner.

The fleet in question is a set of Claude Code agents in tmux panes on one machine, each handling recurring work in its own lane, with one human, me, routing tasks between them. This week its dispatcher is still `tmux send-keys -t fleet:coder_3 '<task>' Enter`. Reads come back through `tmux capture-pane -p`, a text scrape of whatever is on screen at the moment I poll it. Shared state between bots lives in tmux's global environment, and a poller holds leases so two bots don't grab the same job.

None of that changed this month, and I looked hard at whether it should.

The decision: **I'm not adopting ACP, a Hermes-style gateway, or LangGraph's checkpointed-state model for this fleet right now.** This is a judgment call under current constraints, not a verdict on the protocols.

## The convergence, stated precisely

Three things showed up when I read these projects' own READMEs on 2026-07-12, in roughly the same window, and it's worth being exact about who did what, because the framing "everyone is converging" is doing a lot of work if you don't check it.

1. OpenHands adopted ACP (Agent Client Protocol). ACP is Zed's protocol (`zed-industries/agent-client-protocol`), Zed originated and ships it. OpenHands did not build ACP; it added support so OpenHands can run any ACP-compatible agent through a common client interface.
2. Hermes Agent (NousResearch's agent framework, not their Hermes LLM line) standardized on a gateway as its control plane, per its README: a single addressable front door for routing requests to agents, rather than each integration inventing its own transport.
3. LangGraph's pitch is durable, checkpointed state: the graph's execution state gets persisted so a run can resume after a crash or a long pause, rather than living only in a process's memory.

Three different mechanisms (client protocol, gateway, checkpoint store) all pointed at the same problem: give a fleet of agents a structured, addressable interface instead of an ad hoc one. That's a real trend and I take it seriously. One adjacent release from the same window, Agent Canvas (shipped 2026-06-16, a visual multi-agent workspace), is not part of the control-plane story; I name it only to avoid conflating it with the three above.

## What's actually running here

My side of the comparison, plainly:

- **Dispatch**: `tmux send-keys` into a named pane. No request schema, no method names, just a string typed at a prompt.
- **Read**: `tmux capture-pane`, a static scrape of the pane's current text. No acknowledgment that the bot received anything, no cursor or offset tracking; I re-read the whole visible buffer and diff it myself.
- **Shared state**: tmux's global environment, readable by any pane's poller.
- **Scheduling**: a poller process holding leases per bot to avoid double-dispatch.

That's the whole control plane. It is, by any protocol-design standard, unstructured.

## Why I'm not switching

1. **Send-keys dispatch stays**, because it costs zero integration work against a terminal-native tool (Claude Code) that already speaks to a TTY; the tradeoff is that dispatch is a bare string with no schema, so a malformed task just sits there silently instead of failing loudly.
2. **capture-pane stays a scrape**, because it needs no cooperation from the agent process, it works against anything that prints to a pane; the tradeoff is exactly what it sounds like: no ack, no cursor state, and I can miss a line that scrolled past between polls.
3. **tmux global env stays the state store**, because every pane already has read access to it with no extra daemon; the tradeoff is one flat namespace with no ownership boundaries, which is precisely how it gets poisoned.
4. **Poller leases stay the scheduler**, because they're twenty lines of Python against a filesystem lock, not a service; the tradeoff is no retry semantics, no backpressure, no queue depth visibility beyond what I query by hand.
5. **No checkpointed durable state**, because nothing in this fleet runs long enough between my check-ins to need crash-resume; the tradeoff is that a bot that dies mid-task loses that task's progress, not just its process.
6. **No standardized gateway**, because there's one human (me) routing work, not multiple services that need a common front door; the tradeoff is that if I ever add a second orchestrator or a partner integration, every one of these five decisions gets re-litigated at once.

## The measurement

Before I interpret any of this, here's what I pulled from the fleet's logs and a manual message audit on 2026-07-12:

| Metric | Value |
|---|---|
| Weekly token volume | 645M (93% bot-originated) |
| Messages audited | 812 |
| Coordination overhead | ~40% |
| Review backlog | 15 items, oldest 23 days |
| Distinct state locations | 12+ |
| tmux env-poisoning incidents | 3 |

Read flat: bots generate the overwhelming majority of traffic in this fleet (93% of 645M weekly tokens), and a manual pass over 812 messages found roughly 40% of them doing coordination work, retries, status pings, lease negotiation, rather than task work. The review backlog sits at 15 items with the oldest at 23 days, which is a queue that isn't draining. State for the fleet is scattered across 12+ locations (tmux env, poller lease files, per-bot log tails, a couple of ad hoc JSON files), and I've had three incidents where one bot's write to shared tmux env corrupted a variable another bot depended on.

None of these numbers indict tmux specifically. A gateway or a checkpoint store doesn't reduce coordination overhead by itself; that number is a function of how much cross-bot negotiation the *work* requires, and a typed protocol just makes that negotiation cheaper to author and harder to get silently wrong. The 40% coordination overhead is the number most likely to still be 40% after a protocol swap, just spent on structured messages instead of scraped ones.

The scarce resource in this fleet isn't message throughput or protocol expressiveness. It's my decide-bandwidth, the rate at which I can make a bounded judgment call on a bot's behalf and move on to the next one. A control-plane protocol optimizes message structure. It doesn't touch decide-bandwidth directly, and right now that's the binding constraint, not schema drift or ack loss.

## What the protocol buys, concretely

ACP's requests are typed and addressed to a session, roughly:

```json
{
  "jsonrpc": "2.0",
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_9f2a",
    "prompt": [{ "type": "text", "text": "run the migration script" }]
  },
  "id": 42
}
```

My equivalent:

```
tmux send-keys -t fleet:coder_3 'run the migration script' Enter
```

No session id, no request id, no typed content block, no response correlation; the "response" is whatever text shows up in the pane the next time I scrape it. The ACP shape buys request/response correlation, structured content, and a session boundary that a gateway or a second client can reason about without reading my shell history. What it costs is a client implementation, a server implementation on the agent side, and a schema I now have to keep in sync across both.

## Why this matters beyond the fleet

The cost of the unstructured side isn't abstract, it lands as debugging time. A scrape with no ack means a task that died silently looks exactly like one still running, so I learn about the failure by noticing output that never arrived, not from an error. One flat namespace with no ownership boundaries means one bot's write can corrupt a variable another bot depends on, and the symptom surfaces somewhere unrelated to the cause. Three of those env-poisoning incidents are in the table above, and those are only the ones I traced back. Every one of them was time spent reconstructing, by hand, what a typed request and a per-session state boundary would have handed me at the call site. Today that bill is small enough to absorb. The failure mode I'm watching for is the one where absorbing it stops being possible, where a growing share of my work is misdiagnosing silent failures a typed protocol would have made loud.

## What could be wrong with this measurement

The 40% coordination-overhead figure came from a manual read of 812 messages, which is a sample, not the full 645M-token population, and my definition of "coordination" versus "task work" was a judgment call I made alone rather than a labeled taxonomy. The three env-poisoning incidents are the ones I caught; a flat shared namespace makes silent corruption plausible, so the true count could be higher. The 23-day-old backlog item might reflect one stuck ticket rather than systemic drift; I didn't check whether it's an outlier before including it.

## When I'd switch

Two triggers, not a timeline. First: a second agent backend joins any lane of this fleet. At that point send-keys stops being a private shorthand and becomes an integration surface, and a typed protocol earns its cost. Second: a fourth env-poisoning incident, on top of a fix I've already shipped for the first three. A recurring failure in the same mechanism after a patch is evidence the mechanism, not the instance, is wrong. Absent either, this stays boring on purpose.
