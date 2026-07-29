# Example Scripts

Real production scripts. They predate parts of the guide and may contain errors —
they show shape and phrasing, not rules. When the guide and an example disagree,
the guide wins.

| # | Script | Type | Shows |
| --- | --- | --- | --- |
| 01 | [Omni Leads — Debt Consolidation](01-omni-leads-debt-consolidation.md) | Outbound qualification → live transfer | Four-way pickup classification, mid-flow arithmetic on answers, hidden questions, custom-output questions (enum, ISO 8601), bilingual transfer routing, webhook before `end_call`, digits-to-words TTS rule |

## Adding an example

1. Add `NN-short-slug.md` in this directory.
2. Open with a one-paragraph note on shape and notable techniques, and a warning
   if the script predates current conventions.
3. Include every layer that exists for it: Voice Prompt, Intro, Questions, callback
   Intro, after-hours Intro.
4. Add a row above.

Keep the whole script, warts included. The errors are useful — they show what
actually ships and what to watch for in review.
