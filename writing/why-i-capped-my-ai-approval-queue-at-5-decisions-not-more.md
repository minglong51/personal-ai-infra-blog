---
title: "Why I capped my AI approval queue at 5 decisions, not more"
date: 2026-08-24
readingMinutes: 6
summary: "Human approval is a capacity-constrained service, so I cap my remote AI decision docket at five and drain the backlog in focused review sessions."
cover: img/posts/why-i-capped-my-ai-approval-queue-at-5-decisions-not-more-cover.png
coverAlt: "A dense field of agent signals narrows through a filter into one decision card held by a person."
syndicate: [x]
canonical: https://www.minglongpan.com/writing/why-i-capped-my-ai-approval-queue-at-5-decisions-not-more
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/why-i-capped-my-ai-approval-queue-at-5-decisions-not-more). That version is canonical and may be updated; this file is a point-in-time mirror.

On 2026-08-13, my personal knowledge system had this approval backlog:

| Queue category | Items |
|---|---:|
| Actionable | `31` |
| Withheld by the privacy boundary | `26` |
| Safe to expose in the shared feed | `5` |
| Remotely executable | `3` |

The system separates temporary working notes from durable knowledge. Changes that verify or promote a note require human judgment and an approval receipt. Telegram surfaces a small remote docket, while higher-context review happens in a working session.

I decided to keep that remote docket capped at five decisions. I tested the apparent fix for the larger backlog: expose the withheld items as redacted rows so I could approve them by index. Validation showed that the change would weaken several system boundaries.

Human approval is a capacity-constrained service. Once a queue grows faster than its operator can form judgments, adding approval buttons increases nominal throughput without increasing reviewed throughput.

## The evidence

The five-item surface represented only the shared-safe portion of the backlog. Most pending decisions concerned material that the shared Telegram feed was not allowed to identify, and some visible items still required actions unavailable through Telegram.

The eventual drain was recorded as two review sessions:

```text
First review session:  queue 31 -> 14
Second review session: queue 14 -> 0
```

Two high-context review passes cleared the queue while preserving the remote approval boundary.

There is useful outside evidence for treating attention as the constrained resource. Alex Wauters’s [Scale X permission game](https://scalex.dev/blog/ai-agent-permissions-stats/) reported the following results, with counts and rates sorted within their groups:

| Measure | Result |
|---|---:|
| Individual decisions | `409,000` |
| Game runs | `>40,000` |
| Mean accuracy | `66.3%` |
| Threat share in the game | `~34%` |
| Sessions ending with a negative score | `32.9%` |
| Players approving every prompt | `7%` |

Players missed roughly one threat in three while explicitly watching for malicious commands. I would not call this a controlled study. It was a time-boxed browser game with a much higher threat frequency than ordinary development, so it does not establish a production miss rate. My own queue snapshot is also one day from one personal system. It measures backlog composition and drain mechanics, not approval accuracy.

The product stake is narrower than proving a universal failure rate. If approval is the integrity boundary for durable knowledge, the product must preserve the operator’s ability to understand each decision, even when that means leaving most of the backlog outside the quick-action interface.

## 1. Keep the remote docket small, because attention does not scale with queue depth

The queue contained `31` actionable items, but only `3` could be both disclosed and executed through Telegram. Presenting the rest as additional buttons would have increased the number of apparent choices without adding the context needed to decide them.

My ability to check the source was the bottleneck. I needed to understand why a note remained useful and distinguish verification from administrative cleanup after the system had generated its recommendations.

The accepted tradeoff is latency. Some decisions wait for a focused review session instead of being cleared from a phone. In my view, that is preferable to manufacturing throughput by reducing each decision to a title and an action verb.

## 2. Withhold before rendering, because redaction at the last screen is too late

The shared review feed contains this policy:

```json
"privacy_floor": "shared"
```

The queue generator, which prepares items for review, computes visible and withheld populations separately:

```python
actionable_shared = [
    entry for entry in actionable
    if entry["shared_safe"]
]

"actionable_shared": len(actionable_shared),
"actionable_withheld": len(actionable) - len(actionable_shared),
```

The component that formats Telegram messages receives the shared entries and a count of everything else. It never receives the withheld entries themselves. There is therefore nothing at the presentation layer to replace with a redacted label.

Changing this would require private metadata to cross the disclosure boundary before redaction. That moves the privacy guarantee from a structural property of the feed to a promise made by every downstream renderer.

The accepted tradeoff is a less complete remote view. Telegram can report that work is waiting without identifying that work. I lose the convenience of reviewing every item from one surface, while retaining a simpler claim about where sensitive metadata can travel.

## 3. Refuse context-free rows, because an approval object must identify what it authorizes

The shared review format requires every item to carry both a title and the source file it refers to:

```json
"required": [
  "id",
  "title",
  "canonical_path",
  "domain",
  "action"
]

"canonical_path": {
  "type": "string",
  "description": "Every item MUST link back to a canonical Markdown path."
}
```

The generated review feed is also tracked in version control:

```text
$ git ls-files generated/dashboards/review-queue.json
generated/dashboards/review-queue.json
```

A redacted row has no valid representation in this format. Giving it a real title or path would disclose the metadata the privacy boundary is meant to withhold. Giving it a placeholder would sever the approval from the source object. Because the generated feed is tracked, emitting private paths would also persist them outside the private tier.

The accepted tradeoff is that some work cannot be represented in the shared protocol at all. I could add a parallel opaque identifier system, but the operator would then approve an identifier without seeing the underlying evidence. That would create a separate authorization protocol with weaker context.

## 4. Enforce the cap during execution

The component that applies approved decisions rejects an oversized mapping between displayed choices and underlying items:

```python
mapping = docket.get("mapping")
if not isinstance(mapping, dict) or not 1 <= len(mapping) <= 5:
    raise ValueError("docket mapping is empty or oversized")
```

The notification and execution components share the same limit and the same narrow set of permitted remote actions:

```python
MAX_DOCKET_CARDS = 5
TELEGRAM_EXECUTABLE_ACTIONS = {"extend_ttl", "archive"}
```

The limit is part of the authorization mapping rather than merely the number of cards rendered. The mapping is bound to the docket, and each item is bound to the content that existed when the request was produced. If the underlying note changes, the pending decision is no longer the same decision.

The narrow action set matters as much as the docket size. Extending the useful life of a note and moving a settled note into a recoverable archive are bounded administrative actions. Declaring a claim verified or consolidating it into durable knowledge requires broader judgment, so those actions remain outside Telegram.

The accepted tradeoff is an intentionally incomplete remote executor. It handles small, recoverable maintenance decisions and routes the rest to a session where source material can be inspected.

## 5. Drain in context, because batch review can share evidence without collapsing judgment

The queue moved through this sequence:

```text
31 -> 14 -> 0
```

That result came from focused review sessions that reused context across related notes, checked current sources, and separated routine retention decisions from claims requiring re-verification.

The chat notification schedules the work. It tells me that a bounded set of decisions is ready and routes the remainder to the review environment that can supply the necessary context. Expanding it into a universal control plane would optimize the visible interface while leaving the actual bottleneck untouched.

The accepted tradeoff is bursty throughput. The queue may accumulate between review sessions, then fall sharply when related decisions are examined together. That is less visually tidy than continuous inbox clearing, but it better matches how the work is performed.

## When I would use this design

I would cap an approval surface when one operator owns the queue, the actions mutate durable state, the items require heterogeneous context, and privacy rules prevent that context from traveling with every notification. The cap should be enforced by the executor and paired with a clear route to deeper review.

For homogeneous, reversible work that can be decided from complete evidence on one screen, I would use deterministic policy, recoverable execution, and audit receipts instead of asking a person to approve a longer list.
