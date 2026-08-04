---
name: explain
description: Explain a topic, or reasoning/decisions Claude made earlier in the conversation, as a ~750-word opinionated article with short code snippets. Invoke as /tbright:explain <sentence describing what to explain>. Use when the user wants a written-up explanation rather than a quick answer.
---

# tbright-explain

Write a short, opinionated technical article explaining the topic given in the arguments. The arguments are a sentence like "Explain why we split the searcher into two phases" — they may point at a general topic or at reasoning you did earlier in this conversation. If they point at earlier reasoning, mine the conversation for the actual problem, the alternatives weighed, and the solution chosen; the article explains *that*, not a generic version of it.

Output goes directly in the chat reply. Nothing is written to disk.

This is requested long-form content: write it in normal, full prose regardless of any terse/compressed communication mode active in the session. (Conversation text around the article still follows the active mode.)

## Format constraints

- **~750 words of prose** (700–800 acceptable). Code snippets do NOT count toward the budget, but keep each snippet short: 5–25 lines.
- Snippets in the language the topic lives in (default: the language under discussion in the conversation; else Go).
- Markdown headers between sections. No preamble before the first paragraph — open cold.

## Style (mandatory — this is the point of the skill)

Model the article on "The Repository Pattern with Rich Domain Models" structure and voice:

1. **Open with the pain.** First paragraph names the common naive approach and where it breaks: "Most X handle Y the same way: ... It works, until the rules get interesting." Pick ONE concrete running example domain where the rules get interesting fast, and thread it through every section.
2. **State the division of labor up front.** One crisp sentence per concept, then the failure mode of confusing them: "One governs behavior, the other governs movement. Confuse the two and you get ..."
3. **Show the naive version first.** A short snippet of the anemic/obvious approach, then a one-liner demonstrating exactly what goes wrong, with a pointed comment:
   ```go
   flight.Status = "DEPARTED" // with zero crew assigned, at a gate, in the past
   ```
4. **Then the fix, and after each snippet: "What this affords you:"** — concrete guarantees (what becomes impossible, where the single source of truth now lives), never adjectives like "cleaner" or "more maintainable".
5. **Each concept answers exactly one question.** Name the question explicitly ("The aggregate answers: what has to be true at save time?"). If the explanation has N concepts, it has N questions, all distinct.
6. **Say what each piece deliberately does NOT do.** ("Note what the model knows nothing about: databases, rows, transactions.") Negative space is part of the design.
7. **One honest tension.** Every real design has a wrinkle. Give it its own short section, admit it plainly, show the pragmatic escape hatch, and name it for what it is ("It's a trust boundary, documented as such — a small price for ..."). Never pretend the solution is free.
8. **Close with "How the ideas lock together."** Restate each concept's one question in a sentence, then what sags if you skip each piece, ending on the payoff in vivid operational terms ("the thing that actually matters at 4 a.m. ...").
9. **Voice:** direct, opinionated, second person, short declarative sentences. No hedging, no "arguably", no survey of alternatives you aren't recommending. If an alternative was considered and rejected in the reasoning being explained, state it and say why it lost in one sentence.

## Process

1. Parse the argument sentence; decide whether the subject is a general topic or reasoning from this conversation. If reasoning: extract the real problem, constraints, rejected options, and chosen solution before writing.
2. Choose the running example (prefer the actual code/domain from the conversation over an invented one).
3. Draft the article per the style rules; write the snippets first, then the prose around them.
4. Count the prose words (excluding code blocks). Outside 700–800 → cut or expand, favoring cuts to hedges and restatements.
5. Reply with the article only — no meta-commentary before or after it.
