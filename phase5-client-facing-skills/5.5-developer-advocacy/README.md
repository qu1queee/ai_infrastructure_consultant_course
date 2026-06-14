# 5.5 Developer Advocacy Craft

## Learning Objectives

- Build demos and code examples that are concise, copy-pasteable, and show a real performance or cost tradeoff
- Write a technical guide that a developer can follow without prior knowledge of the platform
- Design and run a hands-on workshop for 10–50 developers, with working exercises
- Structure and deliver a conference talk that includes a live or recorded demo
- Run a structured developer feedback session and distill findings into actionable product signal
- Collaborate with marketing on developer use case stories without losing technical accuracy

---

## Why This Module Exists

Developer Advocacy is a craft, not a title. The output that makes a DA effective is not "content" in the abstract — it is specific artifacts that help developers ship faster: a working demo they can fork, a guide that answers the exact question they were stuck on, a benchmark script they can run against their own workload. This module is about building those artifacts deliberately, not by instinct.

---

## Demo Building Principles

A good demo has one job: make a tradeoff legible. The best Token Factory demos are not tours of the platform — they are scripts that let a developer see, in numbers they care about, why a choice matters.

### What makes a demo land

- **Runnable in under 5 minutes.** If it requires infra setup, it will not be run. Ship a Colab, a Replit, or a single `pip install` + API key.
- **Shows a real comparison.** TTFT at batch_size=1 vs. batch_size=32. Cost-per-million-tokens at different model sizes. A demo that only shows "it works" proves nothing.
- **Produces a number the developer can quote.** "200ms TTFT at 100 QPS for Llama 3.1-70B" is shareable. "Performance was good" is not.
- **Fails gracefully.** Show what happens at the edges. A demo that shows retry behavior or rate limit handling is more trustworthy than one that only shows the happy path.

### Demo repository structure

```
token-factory-demos/
├── README.md           ← one-paragraph pitch + requirements
├── 01-basic-chat/      ← simplest possible working integration
├── 02-streaming/       ← streaming completions, TTFT measurement
├── 03-benchmark/       ← TTFT, throughput, cost across models
├── 04-rag-pipeline/    ← RAG with embedding + Token Factory inference
└── 05-cost-compare/    ← Token Factory vs. self-hosted vLLM cost model
```

Each subdirectory: one `README.md`, one runnable script or notebook, no external dependencies beyond the API key.

### Performance metrics that matter to developers

| Metric | What it measures | Why developers care |
|---|---|---|
| TTFT (time-to-first-token) | Latency before streaming begins | Perceived responsiveness in chat UIs |
| TPOT (time-per-output-token) | Token generation speed | Completion speed for longer outputs |
| Throughput (tokens/sec) | System capacity | How many concurrent users the platform handles |
| $/1M tokens | Cost efficiency | Budget planning and model selection decisions |
| P99 latency | Tail behavior | SLA and reliability guarantees |

Build a benchmark script that outputs a table of these metrics. Developers will run it and share the results.

---

## Technical Content Creation

### The guide that actually gets read

Most technical guides fail because they start with architecture instead of a working outcome. The structure that works:

```
1. What you'll build (one sentence + a code snippet showing the result)
2. Prerequisites (minimum viable list — be ruthless)
3. Step 1: [smallest possible working thing]
4. Step 2: [add the next piece]
5. ...
6. What to do next / where to go deeper
```

Never start a guide with "Overview of Token Factory" or a system diagram. Start with `curl` or a Python snippet that works. The developer's question is "can I do the thing?" — answer that first.

### Blog post structure for performance content

1. **The claim** (specific, falsifiable): "Token Factory serves Llama 3.1-70B at 3× lower cost than self-hosted vLLM on equivalent GPU hours"
2. **The methodology** (how you measured it, what assumptions you made)
3. **The numbers** (table or chart — don't make readers parse prose for data)
4. **The tradeoffs** (when this claim holds and when it doesn't — this builds trust)
5. **The code** (reproducible benchmark script, linked on GitHub)

Credibility comes from showing the methodology and tradeoffs, not from the claim alone. Developers will replicate your benchmark and come back if your numbers are honest.

### Content types and their purposes

| Format | Best for | Time investment |
|---|---|---|
| Getting-started guide | Onboarding new users to Token Factory | High — build once, drives ongoing adoption |
| Performance benchmark post | Driving awareness, developer sharing | High — requires real measurement |
| Integration guide (LangChain, LlamaIndex, etc.) | Meeting developers in their existing stack | Medium |
| Cookbook / recipe | Answering a specific "how do I" question | Low — narrow scope, high value per hour |
| Workshop | Hands-on adoption at events | High — but reusable across events |
| Short demo video | Conference, social, X/Twitter | Medium — 2–5 minute target |

Prioritize guides and benchmarks first. They compound: a good benchmark post gets shared; each share is a demo you didn't have to give.

---

## Workshop Design and Delivery

### Anatomy of a hands-on workshop

A 90-minute workshop for 10–50 developers at a conference or meetup:

```
0:00 – 0:10   Why this platform / what you'll build (keep short — action beats slides)
0:10 – 0:30   Exercise 1: Get your first response from Token Factory (everyone runs it)
0:30 – 0:50   Exercise 2: Measure TTFT and throughput (benchmark script)
0:50 – 1:10   Exercise 3: Build a simple RAG pipeline on top of Token Factory
1:10 – 1:20   What to do next / where to get help / Q&A
1:20 – 1:30   Buffer (always needed)
```

**Rules for workshop exercises:**
- Every exercise must be completable in the allotted time by a developer who has never used the platform. Test this.
- Pre-provision API keys or use a shared key with rate limit headroom. Nothing kills a workshop like a sign-up flow during the session.
- Have a Colab backup for every local-environment exercise. Assume 30% of attendees will have environment issues.
- Publish exercises publicly before the workshop. Attendees who arrive early will start — this is good.

### Feedback collection during workshops

The highest-value feedback comes from watching, not asking. During exercises:
- Note which step causes the first pause or question. That is a documentation gap.
- Note the first error message attendees hit. That is a UX gap.
- Note questions attendees ask each other (not you). Those are concepts that need better docs.

After the workshop: a 3-question form max. "What was the most useful thing?" / "What was confusing?" / "What would you build with this?" More than 3 questions → response rate drops to noise.

---

## Conference Talking and Live Demo

### Talk structure for a technical DA talk (20–30 min)

```
[2 min]  Hook: a specific problem your users have, with a real number
[5 min]  Context: why this problem is hard / what the common approaches miss
[15 min] The demo: live or recorded, showing the solution with real metrics
[5 min]  What to do next: GitHub link, docs link, your contact
[3 min]  Q&A
```

The demo is the talk. Everything else is setup and landing. Do not let slides eat the demo time.

### Live demo survival rules

- **Always have a recorded fallback.** Record the demo the night before. If the live demo breaks (it will, eventually), switch to the recording without apology.
- **Hard-code the API key for the demo.** Rotate it after. Do not type credentials live.
- **Show real latency.** Artificial speedups or pre-cached responses destroy credibility with technical audiences.
- **Narrate what you're measuring.** "I'm running this with batch_size=1 — watch the TTFT number in the top right."
- **Leave the terminal visible.** Developers trust code they can see. Slides over a hidden script are a red flag.

### What makes a demo memorable

Developers remember a demo if it showed them something they didn't expect. The most shareable demos are usually benchmarks that produced a surprising number — not tours of a UI.

---

## Developer Feedback Loop

### Your role as a signal aggregator

As a DA, you are the closest person to developers who are actually building with Token Factory. That is a privileged position — product and engineering have limited direct access to that signal. Your job is to convert raw developer frustration, confusion, and excitement into product signal that engineering can act on.

### What "good feedback" looks like to a product team

Bad: "Developers find the docs confusing."
Good: "During three workshop sessions, the same step — connecting a custom model endpoint — caused every attendee to pause. The current docs assume knowledge of the model ID format that isn't documented. Three developers asked if this was a bug."

The difference is specificity: which step, how often, exact error or question, user assumption that broke down.

### Feedback routing

| Signal type | Route to | Format |
|---|---|---|
| Repeated onboarding failure | Docs team | Specific step + frequency + error message |
| Missing API feature (high request volume) | Product | User quote + use case + frequency across developers |
| Pricing/cost confusion | Product + Marketing | The specific comparison developers are making and why |
| Bug or unexpected behavior | Engineering | Reproducible script + expected vs. actual behavior |
| "I chose a competitor because..." | Product + Leadership | Exact quote + context + what would have changed their decision |

Keep a running log. Monthly, write a 1-page "what developers are telling us" summary. This is one of the highest-leverage outputs a DA produces.

---

## Working with Marketing

The tension: marketing wants simplified stories; you know the technical nuance that makes the simplified stories wrong. The resolution: write the technical version first, then simplify together.

**What to contribute:**
- Real developer quotes from workshops and feedback sessions (get permission)
- Benchmark numbers with methodology (so marketing can cite them without misrepresenting)
- Use case narratives: "a team building a customer support agent saw X improvement with Y approach"

**What to push back on:**
- Claims that can't be reproduced (a developer will try and tweet the failure)
- Simplified benchmarks that omit the methodology
- Use cases that don't match how developers actually build

Your credibility with the developer community is the long-running asset. Don't trade it for a launch-week headline.

---

## Key Takeaways

- Good demos show a tradeoff with real numbers. They are runnable in 5 minutes and produce a number the developer can share.
- Technical guides start with a working outcome, not architecture. The developer's question is "can I do the thing?"
- Workshops live or die on exercise design. Test every exercise against the time box before the event.
- Live demo survival rule: always have a recording. Never fake latency.
- Your most valuable output to the company is structured product signal — specific, reproducible, routed correctly.
- Your credibility with developers compounds over time. Prioritize accuracy over marketing alignment when they conflict.
