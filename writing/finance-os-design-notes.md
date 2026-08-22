---
title: "finance-os: a decision system, not a stock picker"
date: 2026-07-24
readingMinutes: 3
summary: A personal investment platform that scores a ~70-name universe on 11 LLM-graded dimensions, blends probability-weighted scenarios, and treats the kill trigger as a first-class object — the design notes.
syndicate: [substack, x]
canonical: https://www.minglongpan.com/writing/finance-os-design-notes
---

> Originally published at [minglongpan.com](https://www.minglongpan.com/writing/finance-os-design-notes). That version is canonical and may be updated; this file is a point-in-time mirror.

finance-os is the largest private system I run. It is not a stock screener and it is definitely not a trading bot. It's a *decision system*: infrastructure for holding an explicit thesis about the world, scoring companies against it honestly, and — the part most investment tools skip entirely — knowing in advance what would prove you wrong. This note covers the architecture. Positions, sizes, and trades stay private by design.

## The thesis is data, not vibes

The system is organized around an explicit, multi-year thesis (mine is the 2025–2030 AI buildout). The thesis is written down as a structured spec — drivers, assumptions, and for each driver a **kill trigger**: an observable condition that would invalidate it. This is the load-bearing choice. Most investment writing is thesis *promotion*; the system forces thesis *accounting*. Every name in the universe exists only in relation to a driver, and every driver carries its own execution condition. When a trigger trips, the system doesn't suggest re-evaluation — it schedules the exit conversation. Counter-thesis is a first-class input, not a mood.

## The universe and the 11 dimensions

The coverage universe is deliberately small — around 70 names — because every name gets scored on **11 LLM-graded dimensions**: capex tier, hyperscaler exposure, management commentary quality, execution risk, and so on. An LLM grades each dimension from structured inputs (filings, transcripts, EDGAR data), and the scores compose into a position view through **probability-weighted scenario blending** — bull/base/bear cases with explicit probabilities, not a single point estimate.

Two disciplines keep the LLM honest:

1. **Structured inputs only.** The model never reads news; it reads documents I can point to. Every score links back to its source passages, so a grade is auditable, not oracular.
2. **The shadow-trade log.** The system records what it *would* have done and scores itself later. Recommended non-actions are logged with the same care as actions — the explicit goal is measuring whether the system's restraint beats its activity. Most tools measure picks; this one also measures passes.

## Monitors are exception-driven

A daily monitor runs the thesis-driver checks and kill-trigger watches, and pushes to my Telegram — but only on state *changes*. A quiet day produces no message. This is the same discipline as the rest of my systems: an alert that fires every day gets ignored by day four, so the only acceptable alert is one that means "something you care about changed." Tier-aware sizing means the monitor also knows which positions are allowed to be loud: a core position's trigger outranks a satellite's every time.

## The stack, and why it's boring on purpose

~7,500 lines of Python on SQLite, an Alpaca paper broker for rehearsal, yfinance and SEC EDGAR for data, and a Telegram bot as the only notification surface. No real-time anything, no dashboard I could lose an afternoon to. The boring stack is a feature: the system's job is to change a handful of decisions per quarter, and every piece of it is pointed at that. The moment a tool wants daily attention, it has become the thing it was supposed to protect me from.

## What I'd tell someone building their own

- Write the kill trigger when you write the thesis, not after the drawdown.
- Score few things well; the universe you can hold honestly is smaller than you think.
- Log the non-actions. Your hit rate on passes is the number nobody tracks and the one that compounds.
- LLM grading is a clerk, not an oracle — structured inputs, auditable scores, and a shadow log to check the clerk's work.
