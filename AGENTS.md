# AGENTS.md — Course Generation Guidelines

This file defines how AI agents should generate content for this course. Read it before creating or modifying any course material.

---

## Course Goal

Build the skill set required to work as a Solutions Architect or Customer Engineer at an AI cloud company, and as a consequence, operate as an independent consultant for the same customer ecosystem.

**Target roles:** AI/ML Specialist Solutions Architect · Customer Engineer · Key Customers Solutions Architect · Principal Solutions Architect

---

## Module Structure

Every module follows this layout:

```
phaseN-name/
└── N.M-module-name/
    ├── README.md        ← objectives, concepts, study notes
    ├── exercises/       ← hands-on tasks (one file per exercise)
    └── resources.md     ← curated reading, docs, and videos
```

### README.md (per module)
- Start with a **Learning Objectives** section (3–5 bullet points)
- Cover concepts with enough depth to be useful, not just a pointer to external docs
- Use tables, code blocks, and short sections — avoid long prose paragraphs
- End with a **Key Takeaways** section

### exercises/
- Each exercise is a standalone markdown or code file
- Exercises must be hands-on — no pure reading exercises
- Include expected output or success criteria so the learner knows when they're done

### resources.md
- Only curated, high-quality links — no padding
- Group by type: Documentation, Articles, Videos

---

## Excalidraw Diagrams

Use diagrams **sparingly** — only for concepts that are genuinely hard to grasp from text alone.

**Approved use cases:**
- Multi-node GPU cluster topology (InfiniBand, NVLink)
- Kubernetes GPU scheduling and pod-to-node mapping
- Distributed training data flow (data parallelism vs model parallelism)
- Full reference architecture (training + inference stack)

**Skip diagrams for:** procedural steps, CLI workflows, configuration examples, anything a code block can express clearly.

---

## Content Principles

- Write for someone with a software engineering background who is new to ML infrastructure
- Prefer concrete examples over abstract explanations
- Every technical claim should be grounded in how it shows up in a real customer engagement
- Do not add modules, sections, or exercises that aren't in the curriculum without confirming with the user

---

## Curriculum Index

Defined in `README.md`. Any new phase or module must be added to the index there.

---

## What to Do When Generating a New Module

1. Create the directory structure above
2. Write `README.md` with objectives, concepts, and key takeaways
3. Add at least one exercise in `exercises/`
4. Add `resources.md` with 3–6 curated links
5. Update `README.md` at the repo root if a new phase or module was added
6. Add an Excalidraw diagram only if the module is on the approved list above
