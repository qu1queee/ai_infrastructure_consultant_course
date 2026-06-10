# 5.3 Presales Mechanics

## Learning Objectives

- Understand your role and accountability in a sales cycle as an SA
- Structure a PoC with success criteria that protect your time and the deal
- Distinguish a POV from a PoC and know when to push for which
- Handle the most common technical objections without conceding
- Map stakeholders and brief each one correctly

---

## Your Role in a Sales Cycle

As an SA, you own the **technical win** — not the revenue. The Account Executive (AE) owns the commercial relationship. Your job is to remove technical risk from the deal so the AE can close.

The practical implication: if there is a technical reason a deal should not close (the product genuinely doesn't fit), your job is to surface that early — not to force a fit. Deals that close on false pretenses create churned customers and damage reputation.

### What You Are Accountable For

- **Technical discovery:** understanding the customer's actual problem, not just the stated one
- **Technical validation:** proving the solution works in the customer's environment
- **Enabling the technical champion:** giving your internal advocate the knowledge and materials to sell internally
- **De-risking objections:** handling "it won't work because X" with evidence, not reassurance

### What You Are Not Accountable For

- Pricing and contract terms (AE)
- Executive relationships above the CTO/VP level (AE / overlay)
- Making the customer buy (you can only make the technical case)

---

## The Sales Cycle — SA Touchpoints

```
Qualification → Discovery → Technical Validation → PoC / POV → Close
     ↑               ↑              ↑                  ↑
 SA reviews     SA leads       SA designs         SA executes
 fit signal    tech reqs       PoC scope          + debrief
```

You enter at qualification (is this technically feasible?), drive discovery, and own everything through close on the technical track.

---

## Technical Discovery

Discovery is the highest-leverage activity in a deal. Done well, it surfaces the real problem, identifies the right success criteria, and finds the technical champion. Done poorly, you build a PoC that answers the wrong question.

### Goal of Discovery

Not to present your solution. To understand:
1. What does the customer's current state look like?
2. What is the measurable pain (cost, time, reliability)?
3. What does success look like in 6 months?
4. Who is accountable internally for this problem?

### Questions That Open Deals

- "What happens today when a training job fails partway through?"
- "How long does it take to provision a new GPU node today?"
- "What's the cost of a 1-hour cluster outage during peak training?"
- "Who on your team would be most directly affected if this worked / didn't work?"
- "What have you already tried that didn't work?"

### The Trap to Avoid

Solutioning before discovery is complete. It is tempting to jump to "we can solve that with X" the moment a problem surfaces. Resist. Every problem you hear in the first 20 minutes is a symptom — the root cause takes another 30 minutes to surface.

### Output: Technical Requirements Document

After discovery, write a short document (1–2 pages) covering:
- Current state architecture
- Pain points and their business impact
- Success criteria for a solution
- Technical constraints (existing tooling, compliance, team skills)

Share it with the customer. If they correct it, you didn't finish discovery — go back.

---

## PoC vs POV

### PoC (Proof of Concept)

Answers: **does the technology work?**

- Customer runs your solution against a synthetic or simplified workload
- Success is technical: "it ran," "latency met threshold," "no crashes"
- Easy to win. Easy to lose when the customer finds a different reason not to buy.

### POV (Proof of Value)

Answers: **does the technology deliver measurable business value in our environment?**

- Customer runs your solution on a real workload, in their environment, with their data
- Success is a business outcome: "training time reduced 40%," "infra cost down $200k/yr," "team unblocked in 2 weeks"
- Harder to win. Much harder to lose — if you hit the success criteria, the business case makes itself.

**Always push toward POV framing.** The conversation shift: instead of "let us show you the technology works," frame it as "let us measure the impact on your actual training pipeline." This reframes the evaluation from a technology horse race to a business decision.

---

## Designing a PoC

### Rule 1: Success Criteria Must Be Written, Specific, and Signed Off Before You Start

Verbal agreement on success criteria is no agreement. At the end of a PoC, every stakeholder remembers what they wanted to be true, not what was agreed. Get it in writing — even an email confirmation counts.

Bad success criterion: "we want to see good GPU performance"
Good success criterion: "training throughput ≥ 2.5x baseline on the customer's existing 8-GPU workload, measured with their standard benchmark script"

### Rule 2: Maximum 3 Success Criteria

More than 3 turns a PoC into a free consulting engagement. Customers will add requirements as they go. Every addition is a scope negotiation. If they want to add criterion #4, something else comes off the list.

### Rule 3: 2–4 Weeks Is the Right Duration

- Under 2 weeks: not enough time for the customer's team to validate meaningfully
- 4–6 weeks: deals stall; momentum dies; stakeholders change priorities
- Over 6 weeks: this is no longer a PoC — renegotiate scope

### Rule 4: Define Failure Conditions Upfront

What happens if success criteria aren't met? If you don't define this, a failing PoC drags forever. Define: "If criteria are not met by [date], we will document gaps and decide within 1 week whether to continue with modified scope or close the evaluation."

### PoC Kickoff Checklist

- [ ] Written success criteria signed off by technical lead and economic buyer
- [ ] Timeline with weekly check-ins agreed
- [ ] Customer technical resource identified (who owns their side of the PoC)
- [ ] Environment access confirmed (hardware provisioned, credentials ready)
- [ ] Failure handling agreed
- [ ] SA agrees not to expand scope without written change request

---

## Common Technical Objections

### "We're already using [Competitor X]"

Don't attack the competitor. Ask about gaps: "What's working well with X? What are the friction points?" The gaps are your opening. You are not selling against X — you are addressing a problem X hasn't solved.

### "We need to see it work before we commit"

This is the PoC conversation. Reframe: "Absolutely — let's design a structured evaluation. What are the 2–3 criteria that would give you confidence?" Now you're designing the PoC together, not responding to an objection.

### "Your benchmarks don't match what we saw"

Benchmarks are measured under ideal conditions that never match production. Acknowledge it: "Benchmarks are a baseline — your workload characteristics will affect the numbers. That's exactly what the PoC is for." Then ask: "What specific workload did you run, and what did you see?" Understanding their configuration is more valuable than defending the benchmark.

### "It's too expensive"

Never defend list price. Reframe to TCO: total cost of infrastructure + ops + developer time + opportunity cost of slow training. A cluster that trains 2x faster is worth 2x as many compute hours to achieve the same output in half the time. If the customer has a budget ceiling, that's a configuration conversation, not a "no."

### "Your product doesn't support [feature]"

Two options: (1) it does exist, you need to educate — "let me show you how that's handled via X"; (2) it genuinely doesn't — "you're right, that's a gap today. Let me understand how critical that is for your use case." Never promise a roadmap feature as a close reason unless you have written product commitment.

---

## Stakeholder Mapping

Every deal has four types of stakeholders. Miss one and the deal stalls.

### The Four Roles

**Economic Buyer** — signs the contract. CFO, CTO, VP Infrastructure. Cares about: ROI, risk, strategic fit. Does not want technical details. Speaks to: "this reduces training cost by 30% and de-risks the roadmap."

**Technical Champion** — your internal advocate. A senior engineer or architect who sees the value and will sell internally for you. Without a champion, you lose when you're not in the room. Invest disproportionately here.

**Technical Evaluator** — runs the PoC. The hands-on engineer. Cares about: does it actually work, is it operational nightmare or not. Wins here by being honest, responsive, and making their job easier.

**Blocker** — security, legal, compliance, a competing internal project, or a political stakeholder who benefits from the status quo. Identify early. Don't ignore. Engage directly: "what would need to be true for this to clear your bar?"

### Briefing Each Differently

| Stakeholder | What they care about | Your message |
|---|---|---|
| Economic Buyer | ROI, risk, timeline | Business impact, costs, strategic fit |
| Technical Champion | Technical depth, credibility | Architecture details, roadmap, how to sell internally |
| Technical Evaluator | It works, it's operable | PoC results, operational model, support |
| Blocker | Their concern (security, budget) | Direct engagement, evidence, accommodation |

---

## Post-PoC Debrief

The PoC is not complete when the technical work is done. It is complete when the decision is made.

### Written Summary (Required)

Produce a 1–2 page document:
- Each success criterion: PASS / FAIL / PARTIAL with evidence
- Performance numbers with context
- Any gaps found, with mitigation options
- Recommended next step

Send it before the verbal debrief meeting. Stakeholders read it before the call — the meeting is for questions, not for delivering results.

### The Next Step Must Be Agreed Before the PoC Ends

If the PoC passes: "Based on the results, what's the path to moving forward? Who approves the purchase?" Get a concrete next step with a date.

If the PoC partially passes: "Here's what we proved and where the gaps are. What would need to be true to proceed?" Don't accept "we'll think about it" — it becomes a dead deal.

---

## What Gets SAs in Trouble

- **Overpromising during demo.** If the product can't do it today, don't imply it can. Customers remember.
- **PoC without exit criteria.** It runs forever. The deal stalls. Your time is consumed.
- **Technical win without executive alignment.** Engineers love it, economic buyer never engaged. Deal dies at procurement.
- **Ignoring the blocker.** Security, legal, or a competing internal initiative will kill the deal in the last week if you haven't addressed it.
- **Expanding scope mid-PoC.** Every scope addition resets the timeline and dilutes the success criteria. Push back.

---

## Key Takeaways

- Your job is the technical win, not the revenue. Remove technical risk so the AE can close.
- Discovery before solutioning — always. The first problem a customer names is usually a symptom.
- POV > PoC: frame evaluations around business outcomes, not technical demonstrations.
- Success criteria must be written, specific, ≤3 in number, and signed off before day one.
- Map all four stakeholders: economic buyer, champion, evaluator, blocker. Brief each differently.
- The PoC ends when the next step is agreed — not when the technical work is done.
