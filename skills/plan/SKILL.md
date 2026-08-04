---
name: plan
description: Plan a task before any implementation. Invoke as /tbright:plan <task description>. Enters plan mode, interrogates the user until the task is fully understood, then produces an implementation plan for approval.
---

# tbright-plan

The arguments describe a task the user wants completed. Do not write any code or modify any files. Plan first.

## Process

1. **Enter plan mode** (EnterPlanMode tool) if not already in it.
2. **Explore the codebase** as needed to ground the plan in reality — read the relevant files, don't plan from assumptions.
3. **Interrogate the user.** Ask me questions until you are 95%+ confident that you understand the task. Use AskUserQuestion. Prefer a few rounds of small, sharp question sets over one giant batch; let earlier answers shape later questions.
4. **Produce the plan** and present it for approval (ExitPlanMode). Steps with verification criteria, critical files named, risks and tradeoffs stated.

## Multiple-choice options: sample the distribution

When presenting multiple-choice options, draw them from different areas of the probability distribution — not four variants of the most likely answer. A good option set looks like:

- The **most likely** interpretation/approach (mark it "(Recommended)" if you'd pick it).
- A **plausible alternative** with a genuinely different tradeoff, not a rewording.
- A **long-shot** — the option you'd bet against but that changes everything if the user picks it (different scope, different architecture, "don't build this at all").

If every option you've drafted differs only in detail, you've sampled one peak — throw the set away and widen. The point of asking is to discover which peak the user is on, and that only works if the options span peaks.
