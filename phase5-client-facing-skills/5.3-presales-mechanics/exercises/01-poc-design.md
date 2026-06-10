# Exercise 01 — PoC Design & Objection Handling

## Scenario A — Design the PoC

**Context:** A financial services firm is evaluating GPU infrastructure for ML model training. They currently train risk models on CPU clusters — a full training run takes 18 hours. Their ML team believes GPUs could reduce this to under 2 hours. The deal is €2.4M for a 32-GPU cluster.

The AE has set up a 1-hour discovery call. After the call, their notes say:
- Technical contact: "Mehmet, Senior ML Engineer, has been pushing for GPU infra for 18 months"
- The Head of ML Infrastructure (Priya) is cautious: "We've had failed infra projects before, I want to see hard evidence"
- CFO approved budget contingent on "demonstrable ROI"
- Current training infra: 40-node CPU cluster, custom PyTorch training pipeline
- Security constraint: data cannot leave their private network

**Your task:**

1. Identify which stakeholder is the economic buyer, the technical champion, the technical evaluator, and any likely blockers.

2. You have 30 minutes with Priya (Head of ML Infra) before the PoC starts. What are your 3 most important questions? What are you trying to learn?

3. Write the PoC success criteria. Remember: ≤3 criteria, specific, measurable, signed off upfront.
   - Criterion 1: _______________
   - Criterion 2: _______________
   - Criterion 3 (optional): _______________

4. The data cannot leave their network. What does this mean for the PoC setup? How do you handle this operationally?

5. The AE wants to extend the PoC to 8 weeks to add two more evaluation areas. How do you respond?

6. Draft the 3-sentence email you send to Priya to confirm the success criteria before the PoC starts.

---

## Scenario B — Objection Handling

For each objection, write your response (2–4 sentences). Do not give in. Do not be defensive. Move the conversation forward.

---

**Objection 1:** "We ran your benchmark and got 1.8x speedup on our training job. Your marketing says 4x."

Your response:
_______________

---

**Objection 2:** "We're already using AWS p4d instances for training. We're happy with it."

Your response:
_______________

---

**Objection 3:** "This is too expensive. We can get comparable hardware from [vendor] for 30% less."

Your response:
_______________

---

**Objection 4:** "Our security team flagged concerns about the GPU driver update mechanism. They don't want anything that auto-updates kernel modules."

Your response:
_______________

---

**Objection 5:** (Mid-PoC) "While you're here, can we also test inference serving, model versioning, and integration with our feature store? We want to evaluate the full MLOps stack."

Your response:
_______________

---

## Scenario C — Stakeholder Mapping in a Stalled Deal

A deal has been in PoC for 6 weeks. The PoC passed all 3 technical success criteria. The AE says the deal is "stuck in procurement." The last 3 weekly check-ins have been rescheduled.

**Answer:**

1. What is the most likely root cause of a stalled deal after a technical pass?

2. List 3 questions you would ask the AE to diagnose the stall.

3. Mehmet (your champion) says: "I think there's an internal budget reallocation. Also, our VP of Engineering hasn't been looped in." What do you do?

4. The AE suggests waiting another 2 weeks to see if procurement "comes back." What do you recommend instead?

5. Write the 2-sentence message you send to Mehmet to re-establish momentum without being pushy.

---

## Scenario D — Post-PoC Debrief

The PoC is complete. Results:
- Criterion 1 (training time ≤ 2 hours): **PASS** — achieved 1h 47min
- Criterion 2 (no data leaving private network): **PASS** — full on-prem deployment validated
- Criterion 3 (ops team can manage cluster independently within 2 weeks): **PARTIAL** — team comfortable with day-to-day ops, but driver upgrade procedure needs a 1-day training session

**Write the post-PoC summary:**

1. Draft the 1-paragraph executive summary (for Priya and the CFO). Business framing, not technical.

2. For the PARTIAL criterion: how do you present this without it becoming a reason not to buy?

3. What is the specific next step you propose, with a date?

---

## Reference Answers

**Scenario A:**
1. Economic buyer: CFO; Champion: Mehmet; Evaluator: Mehmet (also runs the PoC); Blocker: Priya (cautious, "failed projects before") and potentially security team
2. What does a "failed project" look like to you? What would need to be true for you to feel confident this is different? What's your timeline pressure?
3. Example: (1) Training time for standard risk model ≤ 2h on provided hardware; (2) Full training pipeline runs end-to-end on on-prem deployment with no data egress; (3) ML team can re-run training independently within 3 days of setup
4. You bring hardware on-site (or access their private cloud), you don't copy data out. Make this explicit in PoC plan.
5. Scope creep: "Those are valid areas to evaluate, but adding them now would change our success criteria and timeline. Let's complete the current evaluation first — we can plan a phase 2 scope."
6. Short, specific, names the criteria, asks for written confirmation

**Scenario B:**
1. Benchmarks use ideal conditions — your workload has different characteristics. Let's look at what drove the 1.8x: batch size, data loading, precision settings. That's what the PoC is for — real numbers on your real workload.
2. What's working well with p4d? What are the friction points — cost at scale, instance availability, training time? I'm not asking you to change what's working; I'm asking where the ceiling is.
3. Happy to compare TCO: total cost including ops, availability, training throughput, and cost per model trained — not just hardware list price. What's your cost per training run today?
4. That's a legitimate concern. The GPU Operator allows you to pin driver versions and control rollout — nothing updates automatically without a change process. Let's set up a call with your security team to walk through the update mechanism.
5. Those are all worth evaluating. Adding them to this PoC would require extending the timeline and rewriting the success criteria — which means restarting the clock. I'd recommend completing criterion 1-3 as agreed, then scoping a phase 2 for the broader MLOps evaluation.

**Scenario C:**
1. Economic buyer never engaged / not aligned; competing internal priorities; budget moved
2. Who is the economic buyer? Have they seen the PoC results? Is there a competing internal project?
3. Request a meeting with Mehmet + VP Engineering: "Would it be useful to do a 30-minute briefing for [VP] on the results? I can prepare a one-pager focused on business impact."
4. Create urgency without pressure: propose a specific date for a decision call with a named agenda
5. "Mehmet, I want to make sure [VP]'s questions get answered before momentum stalls. Can we get 20 minutes on the calendar this week to walk through the results together?"

**Scenario D:**
1. "Based on the PoC, training time was reduced from 18 hours to 1h 47min — an 83% reduction. Full on-premises deployment was validated with no data leaving the network. The ops team is operationally ready with one remaining training session on driver procedures."
2. Frame it as a known, bounded gap with a clear resolution: "The one remaining item — driver upgrade procedure — is a 1-day knowledge transfer session we would deliver before go-live. It's scoped and ready to schedule."
3. "Proposed next step: schedule the driver upgrade training session by [date + 2 weeks], with purchase order initiated by [date]. Does that timeline work for procurement?"
