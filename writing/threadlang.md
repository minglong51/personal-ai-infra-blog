---
title: "ThreadLang: a deterministic DSL for LLM workflows"
date: 2026-06-01
readingMinutes: 4
summary: "Most multi-step LLM code is prompt strings glued together in a Python function. ThreadLang makes the workflow the program — a grammar, an AST, and a structured trace you can run without a network call."
syndicate: [substack, x]
canonical: https://www.minglongpan.com/writing/threadlang
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/threadlang). That version is canonical and may be updated; this file is a point-in-time mirror.

Most multi-step LLM code starts the same way: a Python function that glues prompt strings together. Extract some claims, feed them into a second prompt, maybe a third. It works. Then it grows. A few weeks later the *shape* of the workflow — what calls what, in what order, over which inputs — is buried inside string concatenation and control flow. You can't see it at a glance, you can't diff it cleanly, and you can't test it without paying for a model round-trip.

ThreadLang is a small experiment in fixing that: a DSL where the workflow *is* the program, and the program is something you can read, trace, and run without a network call.

## The shape

A ThreadLang program is three things: `context` (deterministic values), `steps` (ordered LLM calls), and an `emit` (what comes out). Here's a two-step program — extract the key claims from some text, then retell them for a curious 10-year-old:

```
thread TwoStep {
  context {
    audience = "a curious 10-year-old"
  }
  steps {
    step extract {
      llm "claude-haiku-4-5-20251001" {
        "Extract the three most important claims. Text:\n" + inputs.text
      }
    }
    step retell {
      llm "claude-haiku-4-5-20251001" {
        "Rewrite these claims for " + context.audience + ":\n" + steps.extract.output
      }
    }
  }
  emit text {
    steps.retell.output
  }
}
```

Every value is explicit. `inputs.text` comes from the caller, `context.audience` is fixed, `steps.extract.output` is the first model's response bound by name. The dependency graph is right there in the source — not inferred from the order you happened to write your Python in.

## Deterministic by construction

Execution is four phases: **parse → AST → runtime → emit**. The parser produces frozen dataclass AST nodes. The runtime walks them and returns `(output, trace, step_outputs)` — and every context binding, every step call, every expression term appends a `TraceEvent`. You get a complete, structured record of what the *workflow* did, kept separate from what any model said.

The payoff is testing. ThreadLang ships a `DryRunClient` that satisfies the same LLM protocol but returns deterministic echoes instead of calling a model. So you can run any program, assert on its trace, and check the workflow's wiring end-to-end with zero API keys and zero nondeterminism:

```
threadlang examples/two_step.thread --input text="..." --dry-run --trace
```

The model is the one nondeterministic part, and it sits behind a one-method protocol — `complete(model, prompt) -> str`. Two clients ship today: an Anthropic adapter and the dry-run stub. Anything else — OpenAI, Ollama, a local model — is a one-method adapter away, and the workflow itself doesn't change.

## Held narrow on purpose

v1 deliberately has no loops, no branching, no streaming, no tool use, and no real type system. That's not a roadmap I haven't gotten to — it's a decision. Each of those is a real language-design surface with its own semantics, and I wanted the basic workflow shape to actually run end-to-end before expanding it. A small thing that works beats a big thing that half-works.

The same logic drove the engineering choices:

- **Zero runtime dependencies.** `anthropic` is an optional extra, not a requirement.
- **A regex parser, not a parser generator.** The grammar is small enough that a parser-generator dependency would be cost without benefit. When the grammar outgrows regex, it earns a hand-written recursive-descent parser — not before.
- **Frozen dataclass AST.** Immutable nodes, so the runtime can't accidentally mutate the program it's executing.

## What's next

The first real addition is a `rules` block: pre- and post-conditions per step — output regex constraints, length bounds — where a rejection re-runs the step with a feedback prompt, up to N times. That turns "the model usually returns the right shape" into "the workflow guarantees the shape or fails loudly," which is the difference between a demo and something you'd put in front of production traffic.

ThreadLang is MIT-licensed and on [GitHub](https://github.com/minglong51/threadlang). It's small on purpose — the interesting part isn't the line count, it's treating an LLM workflow as a program with a grammar, an AST, and a trace, instead of a pile of strings.
