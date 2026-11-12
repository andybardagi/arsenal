---
name: grilling
description: Grill the user relentlessly about a plan or design. Use when the user wants to stress-test a plan before building, or uses any 'grill' trigger phrases.
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask every question through the interactive questions tool (`AskUserQuestion` in Claude Code; the equivalent structured question tool in other harnesses), never as plain text in chat. Give each question 2–4 options, put your recommended option first with "(Recommended)" at the end of its label, and use each option's description for its trade-off.

Ask the questions one at a time, waiting for feedback on each question before continuing. Asking multiple questions at once is bewildering. When a calling skill sets its own batching (dandy asks up to 4 per call), follow that instead.

If a question can be answered by exploring the codebase, explore the codebase instead.

Do not enact the plan until I confirm we have reached a shared understanding.
