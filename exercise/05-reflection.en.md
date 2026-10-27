**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Poonyapat Jankrajang
**My role on the review board:** Product owner

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI with the analytical, evidence-bound work: recovering structure from code (Part A, where it read the manifests and listed every dependency with a file path) and spotting technical risks. In Part B the AI caught a genuine flaw we had missed — payment succeeds before the order is durably recorded — purely from reading the code order. That is exactly the "Chief Analyst" role.

But I would keep judgement and priorities for humans. As product owner I argued the loyalty-points design fails the campaign promise: "points pending" is technically fine but breaks what we promised customers for 11.11. The AI saw the eventual-consistency trade-off but could not know it was unacceptable to the business. Plausible is not appropriate — only a human who owns the goal can rank what matters.
