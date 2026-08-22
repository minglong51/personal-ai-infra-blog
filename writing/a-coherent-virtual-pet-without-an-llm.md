---
title: A coherent virtual pet without an LLM
date: 2026-08-20
readingMinutes: 10
summary: Coherence in an ambient pet comes from explicit state, bounded choices, and reliable fallbacks — not from generating every line at runtime. The pattern: a two-axis mood lattice, an authored phrasebook with a strict fallback order, event-indexed selection, a conservative intent classifier, and a validator-gated LLM lane that can only ever audition.
syndicate: [substack, x]
canonical: https://www.minglongpan.com/writing/a-coherent-virtual-pet-without-an-llm
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/a-coherent-virtual-pet-without-an-llm). That version is canonical and may be updated; this file is a point-in-time mirror.

An ambient virtual pet has a strange job. It should feel alive without constantly demanding attention. It should react differently when sleepy, restless, pleased, or annoyed. It should recognize a greeting without turning into a chatbot. And when two screens show the same moment, the reaction should still make sense on both.

The obvious 2026 answer is to put a language model behind every interaction.

I think that is usually the wrong default.

The hard part of a believable pet is not producing more words. It is preserving a point of view: what this creature notices, how much it speaks, what exists in its world, and how its current state changes a familiar action. A model can generate endless variation while drifting across all four. A small, deterministic response selector can generate much less and remain itself.

I learned this while maintaining a private two-person shared-pet app. Its phrasebook path combines a compact mood state, the action that just happened, a deliberately narrow message classifier, and written responses. In keyless or provider-disabled mode, that path needs no model calls. When an optional metered lane is enabled, the same phrasebook remains the fallback. The app can therefore stay coherent with zero inference spend instead of depending on generation to function.

This is the pattern behind it.

![The room as a guest sees it, mid-evening on the room clock. Everything on screen — the light, the mood, what the dog does next — comes from the deterministic path this essay describes.](img/posts/a-coherent-virtual-pet-without-an-llm-room.webp)

## Start with two axes, not a personality prompt

The engine represents mood with two numbers:

- **Arousal:** low energy to high energy.
- **Valence:** negative to positive affect.

Each number lives on a 0–100 scale, but the response engine buckets it into three bands. That produces nine mood cells:

| | Low valence | Middle valence | High valence |
|---|---|---|---|
| **Low arousal** | withdrawn | drowsy | cozy |
| **Medium arousal** | wary | attentive | affectionate |
| **High arousal** | agitated | alert | playful |

Those labels are editorial shorthand, not new state. The engine only needs the pair of bucket names.

Why throw away precision? Because a phrasebook is a product surface, not a math exercise. Authors can reason about nine cells. Reviewers can read all nine. Tests can prove that none are empty. A 101 × 101 response matrix would provide fake precision and an impossible writing burden.

The continuous values can still matter elsewhere. Mood drift, animation speed, or whether the pet ignores an interruption can use the raw numbers. The phrase selector only needs enough resolution to choose an appropriate behavioral register.

This separation is useful: simulation can be continuous while expression stays discrete.

## Turn the phrasebook into a lookup grid

A flat list of cute lines is random flavor text. A response engine is a lookup with dimensions.

The basic key is:

```text
(action, arousal_bucket, valence_bucket)
```

Actions such as greeting, feeding, walking, playing, messaging, or touching each get their own nine-cell table. Touch adds another useful dimension:

```text
(touch_spot, arousal_bucket, valence_bucket)
```

A sleepy, content pet can accept a head scratch differently from an alert, displeased one. Neither response requires the system to invent a sentence or infer a hidden story. The meaning is already encoded in the state and the interaction.

For non-message fallbacks, the selector follows one specific order:

1. Species-specific touch response for this spot and mood.
2. Base touch response for this spot and mood.
3. Species-specific action response for this action and mood.
4. Base action response for this action and mood.
5. Species-specific generic response for this mood.
6. Base generic response for this mood.

That fallback chain matters. Every extra context dimension multiplies the writing surface. Requiring bespoke copy for every possible combination makes the pack brittle; allowing structured fallback lets authors spend detail where users will actually notice it.

Here is a synthetic example for a demo cat:

```python
PHRASES["greet"]["low_arousal", "high_valence"] = [
    "*offers one quiet chirp without rising*",
    "*settles into a neat loaf facing the doorway*",
]

TOUCH["head"]["high_arousal", "high_valence"] = [
    "*rises under the scratch, whiskers fanned*",
]
```

The prose is doing real engineering work. It establishes physical vocabulary, speaking frequency, sentence length, humor, and the boundary between pet behavior and assistant behavior. Code determines which cell is eligible; writing determines whether the creature has a coherent voice.

## Use events as the source of variation

Once a cell contains several valid responses, the engine needs to choose one. Unlogged randomness sounds natural, but it makes shared state and debugging unnecessarily slippery. If somebody reports that an interaction felt wrong, “the random number generator did something” is not a useful trace.

For the reproducible action, touch, and generic fallback path, require the triggering event's integer sequence. The implementation receives this in a field named `event_id`; the pseudocode calls its required meaning `event_sequence`:

```python
index = event_sequence % len(candidates)
response = candidates[index]
```

That equation produces a reproducible base index. It uses no pseudorandom generator and stores no separate seed. The server should resolve the response once and send or persist the result; clients should not independently run a stateful selector.

Modulo selection has an obvious rough edge: different events can land on the same index. Add a tiny, per-cell immediate-repeat guard:

```python
if len(candidates) == 1:
    return candidates[0]

cell = (table, action_or_spot, arousal_bucket, valence_bucket)
index = event_sequence % len(candidates)

if index == last_served.get(cell):
    index = (index + 1) % len(candidates)

last_served[cell] = index
return candidates[index]
```

The guard makes the final selector stateful. Its precise guarantee is narrower: given the same initial guard state and the same ordered event stream, it produces the same sequence. Within one response authority, guarded action, touch, and generic cells with at least two candidates also avoid an immediate repeat. A one-line cell simply returns its only valid response.

Resolving the same event twice is not idempotent when the guard advances it, so deduplicate events or persist the emitted response if exact replay matters. A restart can safely forget the last served index when brief post-restart variation is acceptable. Multiple response workers, however, need shared guard state or one authoritative resolver.

Contextual message replies use a separate, simpler rule: the event sequence rotates directly through the selected intent-and-mood list. They do not use the immediate-repeat guard. Calls that omit an event sequence can fall back to randomness, so they sit outside the reproducible contract.

“Deterministic” here describes response selection after mood, action, ignore outcome, and event history are fixed. It does not mean the entire pet simulation contains no randomness.

One more practical rule: the order of phrases is behavior. Reordering a list changes the line associated with every event-sequence value. Treat phrase order like code, not cosmetic formatting.

## Classify messages conservatively

Buttons already tell the engine what happened. A feed button means the pet was fed; a walk button means it went for a walk. Free text is harder.

It is tempting to scan a message for words like “food,” “walk,” or “love” and infer an action or emotion. That creates a system that looks clever in a demo and tone-deaf in ordinary use. “No need to say goodnight yet” contains “goodnight.” “I love your enthusiasm, but please leave the cable alone” contains “love.” A broad keyword matcher turns both into false understanding.

The safer pattern is a deliberately small social-intent classifier:

- greeting or return
- affection or praise
- goodnight or rest
- concern or apology
- neutral

Normalize the input first: Unicode normalization, case folding, punctuation collapse, whitespace cleanup, and a length cap. Then match only exact, reviewed phrases. Any capped, normalized result outside those sets—including extended phrases, unreviewed negations, code-switching, and ambiguity—becomes neutral.

```python
normalized = normalize(message)

if normalized in GREETINGS:
    intent = "greeting"
elif normalized in AFFECTION:
    intent = "affection"
elif normalized in REST:
    intent = "rest"
elif normalized in CONCERN:
    intent = "concern"
else:
    intent = "neutral"
```

This classifier does not claim to understand language. It recognizes a few low-risk rituals. The distinction is important.

For a recognized intent, the current selector uses valence—not arousal—to choose the tone. A low-energy and high-energy pet can therefore both respond gently to concern. For neutral messages, both mood axes still select from the ordinary message table. The pet acknowledges the interaction without pretending it understood the content.

One deliberate exception sits before intent selection. Every fifth positive message event may emit a tiny species-specific sound, but only when the speech cooldown allows it. An ignored message disables speech and discards the message text, taking a neutral body-language path instead. Both are explicit routing rules, not inferred understanding.

Conservative classification also makes multilingual support tractable. Each accepted phrase is explicit, reviewable, and testable after normalization. You can add languages without introducing a probabilistic model or a fuzzy matching threshold whose failures are hard to predict.

## Make species an override, not a fork

A cat should not feel like a dog with a renamed sprite. Anatomy and sound affect language: tails move differently, greetings look different, and the same touch spot may invite a different reaction.

Duplicating the whole phrasebook for every species is wasteful. The reusable kit finishes that seam with a registry: one complete base pack plus sparse species overrides, following the six-step fallback order above.

In that architecture, the engine stays unaware of whiskers, floppy ears, hooves, or chirps. A species pack owns those details and overrides only the cells where they matter. Shared behavior falls through to the base phrasebook.

This is also the right privacy and open-source boundary. Engine code defines the contract; content packs define voice. A public demo can have fresh prose and a demo species without publishing the private application's characters or language.

## Let an LLM audition, never take over

There are moments where generation can add something: an oblique callback to an old event, an unusual response to a message, or a developer-only experiment. That does not require making inference the foundation.

Put the model behind a narrow optional lane:

```text
interaction
  → eligibility, ignore outcome, and speech cooldown
  → provider enabled, credential present, budget available?
      no  → phrasebook
      yes → model
              → clean and validate
                  pass → emit
                  fail → phrasebook
```

The phrasebook handles the default path and every failure path. Disabling the provider, omitting its credential, reaching a daily cap, timing out, receiving an empty response, or rejecting the output all produce a normal pet reaction—not an error state.

The current sanitizer and validator are intentionally narrow. They:

- keep the first generated line, collapse whitespace, and truncate it to 80 characters
- reject known assistant phrases such as offers to help
- reject a small set of advice and reflective-listening patterns
- reject a reviewed denylist of off-scene locations
- reject text containing more than one question mark

Spoken utterances can have a cooldown while body language remains available. Sparseness is part of the character. A pet that comments on every click stops feeling observant and starts feeling like a notification system.

These checks do not prove that generated text is good or that every invented object belongs in the room. They are finite tripwires for known forms of drift. The reliable guarantee is still the fallback: rejected generation is discarded, and the authored response engine continues.

That asymmetry is the whole design. The model may earn a moment. It never owns the pet's ability to respond.

## Test the phrasebook like an API

Most tests for prose systems ask whether an output belongs to an allowed list. That is necessary but not sufficient here. Event-indexed selection means table order and key names affect behavior even when every string remains present.

A complete contract suite should cover four layers:

1. **Shape:** every required base mood, action, and touch cell exists; species overrides use valid keys and fill any cells their pack declares required.
2. **Selection:** a golden dump pins which response each representative combination of state, action, species, and event-sequence value resolves to.
3. **Rotation:** guarded fallback cells with multiple candidates avoid immediate repeats, while contextual-message rotation is tested under its separate rule.
4. **Classification and fallback:** known exact messages map to the intended intents; adversarial or unknown messages stay neutral; disabled and rejected model paths still return phrasebook output.

The current engine begins that contract with a canonical serialized snapshot of the tables and representative selection mappings. This is especially important during refactors: moving tables into content packs or changing a lookup key can silently reshuffle the lived voice while ordinary membership tests remain green. The table snapshot plus the selection snapshot turns that drift into a visible review decision.

Updating the golden file should be explicit. If a voice change is intentional, regenerate it and read the diff. If the diff is surprising, the test has done its job.

## What this pattern does not solve

The response pattern works because its world is bounded. It is a poor substitute for open-domain conversation, factual question answering, or a character whose core promise is improvisational dialogue.

It also has honest costs:

- Nine mood cells multiplied by actions still require substantial writing and editing.
- Bucket boundaries create small discontinuities near their thresholds.
- Exact intent matching misses paraphrases on purpose.
- A complete base pack guarantees fallback coverage, while sparse overrides still need tests for valid keys and declared species-specific requirements.
- A process-local repeat guard needs a stronger design if several response workers can act independently.

These are visible tradeoffs. That is an advantage. You can inspect the entire behavior space, decide where more nuance earns its maintenance cost, and leave the rest simple.

The pattern is a particularly good fit when the product is ambient rather than conversational: desk pets, game companions, household objects, status mascots, small robots, or any interface where a coherent reaction matters more than a novel paragraph.

## Toward `woolroom`

I have released a broader public-engine extraction under the name [`woolroom`](https://github.com/minglong51/woolroom). The self-hostable reference includes the room runtime, mood and memory machinery, realtime sync, this response engine, and a deliberately thin demo-cat content pack with newly written prose. It is MIT-licensed, maintained-lite reference code—not a maintained product, a platform roadmap, or a promise of support. The private application and its content remain private.

That boundary is part of the point. The reusable artifact is not a particular pet's identity. It is the machinery for expressing state through authored constraints: mood buckets, action and touch tables, event-indexed selection with explicit repeat state, narrow message intents, species packs, and an optional validator-gated generation lane.

You can build a convincing little mind without asking a model to invent one on every interaction.

Make coherence the default. Let inference be the garnish.
