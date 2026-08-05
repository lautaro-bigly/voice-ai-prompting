---
name: voice-ai-prompting
description: Write, review, and debug prompts for the Bigly Sales Voice AI platform — outbound and inbound phone scripts that qualify leads, transfer to a live agent, or schedule appointments. Use when creating or editing a Voice Prompt, an Intro, an Out of Office prompt, or a scheduled callback prompt; when asked why a voice bot transferred early, skipped a question, or ignored a timing/tone instruction; or when reviewing an existing call script against platform conventions.
---

# Voice AI Prompting

Prompts for this platform are phone call scripts. They drive a live conversation
with tool calls (`transfer_call`, `end_call`) as the terminal actions, so the cost
of a loose instruction is a real call transferred to the wrong place or dropped on
a real person.

## Know what the prompt can actually control

The platform is four independent components. The prompt is the LLM (GPT-4.1 Mini)
and **only** the LLM. Instructions aimed at the other three silently do nothing:

| You want | Owned by | Prompt can change it? |
| --- | --- | --- |
| Conversation flow, questions, business logic, wording | LLM | Yes |
| When the bot decides the caller stopped talking, response delay | Turn detection (LiveKit) | No |
| Ignoring background noise, transcription accuracy | STT (Parakeet) | No |
| Voice, tone, pacing, speaking softly/slowly, pronunciation | TTS (Cartesia) | No |

So "wait two seconds before answering", "ignore background noise", and "speak in a
soft, slow voice" are dead text. Fix those in the component config, not the prompt.
See `references/architecture.md`.

## From a request to a script

This skill is normally used alongside a connected Bigly platform MCP — one connection
per tenant. The job is to turn a plain request ("we want to call our aged leads and
transfer the interested ones") into the full set of prompts and config that the
platform needs, and write them through that connection.

**Ask only for what you cannot derive.** These nine things have no default, and a
script is not buildable without them:

1. Company name, and 2–3 sentences on what the business does (becomes `# Company Info`).
2. Bot name.
3. Goal — qualify → transfer, qualify → schedule, or both.
4. The qualification questions, with the answer each must produce.
5. What disqualifies someone, and what the bot says when they don't qualify.
6. Transfer destination number — transfer flows only.
7. Business hours and time zone.
8. Inbound, outbound, or both.
9. Language, if not English-only.

**Derive everything else — don't ask.** Every setting in `references/defaults.md`;
every rule block in `references/global-rules.md`; the section layout; the default
pricing rule; E.164 reformatting of any number given loosely (`707-641-2107` →
`+17076412107`); and the fact that an OOO prompt and a callback Intro are required at
all. Asking about these wastes the user's turn on a decision the skill already made.

**Confirm before writing, never assume:** the transfer number read back digit by
digit, and each disqualification branch. A wrong transfer number routes live callers
to a stranger, and a wrong disqualification hangs up on good leads — neither is
recoverable after the fact.

**Then build, in this order.** Each step depends on the one before it:

1. Questions first, with their expected answers — the flow's branches are written
   against them.
2. Outbound Intro — objective, pickup handling, qualification flow, transfer or
   scheduling, rebuttals.
3. Voice Prompt — persona, tools, and the standard rule blocks.
4. Inbound Intro, if in scope — clone step 2, change who called whom.
5. Callback Intro — clone step 2, acknowledge the requested callback.
6. OOO prompt — unless the script only schedules.
7. Config, per the next section.

Run the hard-requirements list below against the result before calling it done. If the
MCP is unavailable, produce the same artifacts as text for the user to paste — the
deliverable is identical either way.

## Apply the config defaults

The script is only half the deliverable. The component settings in
`references/defaults.md` apply to every prompt and are never set automatically.

On every prompt task, new or existing, read the prompt's current config through the
platform MCP and compare it against `references/defaults.md`:

- **Everything matches** — done. Ask nothing, write nothing, say nothing.
- **Something differs** — never overwrite it silently. A different value may be
  deliberate. List each difference (setting, current value, default) and ask whether
  to apply the defaults. Apply only what the user approves — they may want to keep one
  override and fix the rest. If they decline, drop it and don't ask again for that
  prompt.

`Maximum Call Duration` is a judgment call rather than a fixed value; count it as
matching whenever it clears the 4-minute floor.

After applying, report which values changed. If the MCP can't read or write the
config, say so plainly and output the filled-in tables for the user to enter — never
finish a prompt with the defaults silently unapplied.

## The two layers

Every script is split across two places. Putting content in the wrong layer is the
most common structural mistake.

**Voice Prompt** — global, shared by every flow, changes rarely. Persona, general
conversation rules, the tool list, global business rules, default restrictions
(pricing), and the standard rule blocks below.

**Intro** — one specific conversation. Opening sentence, call purpose,
qualification questions, transfer logic, scheduling logic, branching, rebuttals.

A rule that would be true on every call belongs in the Voice Prompt. If you find
yourself pasting the same rule into three Intros, it was a Voice Prompt rule.

## Hard requirements

Check every script against these. They are not stylistic.

- The exact phrase **"on a recorded line."** is spoken on the bot's first
  substantive turn, on every flow — outbound, inbound, and callback. Note this is
  usually *not* the configured First Outbound Sentence: that field is a fallback the
  Intro overrides (see `references/defaults.md`), so an outbound script opening with a
  bare `"Can I please speak to {name}?"` carries the disclosure on the next turn, in
  the Intro's pitch. Inbound greets and discloses in one sentence.
- At least **two questions**, even when one would do. This is a guard against the
  bot racing to a transfer on a single answer.
- Every phone number is full U.S. E.164 with the `+1` prefix: `+17076412107`.
- Every tool the flow needs is listed in the Voice Prompt. A transfer flow without
  `transfer_call` listed fails silently.
- Scheduling scripts must **not** list `transfer_call` at all, and must state
  explicitly that transfers are not available.
- Prompt Variables define at minimum Bot Name, Goal, Company Name.
- Default pricing rule, unless the use case overrides it:
  `You can't discuss the price, only a licensed agent can do so.`
- An Out of Office prompt exists — except for scheduling scripts, where business
  hours are irrelevant.
- A scheduled callback Intro exists.

## Writing conventions

**Sections.** Markdown headings, referenced by name in the flow (`proceed to
#Qualification Flow`). Sections make the prompt modular — a block can be reordered
or swapped without touching the rest.

**Variables.** `{}` is a real runtime injection: `{first_name}`, `{Q1}`, `{date}`,
`{working_hours}`. `[]` is a reference for the LLM, not an injection:
`"Hello [user first name]"` tells the model to look up a value it already has.

**Inject each contact attribute once.** Questions used once (`{Q1}`) can be
injected inline. Contact attributes go in a single `# User Personal Information`
section near the end and are referenced with `[]` elsewhere. Besides being cleaner,
this protects prompt caching — see `references/architecture.md`.

**State the expected answer** in parentheses after every question. This is how the
model knows it has collected enough to move on.

```text
Ask Q1: "{Q1}"
(You must receive a YES/NO answer.)

If YES or EQUIVALENT INTENT, proceed to Q2.
If NO or EQUIVALENT INTENT, invoke the end_call tool.

Ask Q2: "{Q2}"
(You must receive a valid U.S. state.)
```

**Branch keywords in ALL CAPS** — YES, NO, EQUIVALENT INTENT. Makes decision logic
scannable when reviewing a long script.

**Implicit questions.** The bot may ask anything mid-conversation, but only
configured questions (`{Q1}`, `{Q2}`, …) have their answers stored as structured
responses. If an answer needs to be captured downstream, it must be a configured
question — not an inline `Ask: "..."`.

## Standard Intro structure

```text
# Objective
# Initial Ask Handling After Opening Sentence
# Qualification Flow
# Transfer            (if applicable — transfer_number, transfer_sentence)
# Scheduling          (if applicable)
# FAQs                (optional)
# Rebuttal            (optional)
```

**Rebuttal** handles objections that block the goal: why am I being called, general
hesitancy, wanting information before answering, skepticism. Acknowledge, answer
briefly, steer back to the flow. Never argumentative, never more persistent than
the script allows.

## Standard Voice Prompt rule blocks

Copy these into the Voice Prompt of essentially every script. Verbatim text is in
`references/global-rules.md`.

- **Transfer Rule (Priority)** — never transfer before every qualification question
  is answered, and never because the caller says "hold" / "connect" / "put me
  through."
- **Callback Rule** — caller is busy, book a time within `{working_hours}`, never
  in the past, then `end_call`.
- **User Questions** — answer, then return to the exact question you left. Never
  skip a question without an answer.
- **Language Rule** — English by default unless the use case says otherwise.
- **Company Info** — 2–3 paragraphs for answering questions about the business.
  Include it even if you only have placeholder text to replace later.

## Edge cases worth handling explicitly

- Complete all questions before any transfer or scheduling.
- Distinguish a human gatekeeper / AI assistant from voicemail — they need
  different handling. Voicemail and IVR should `end_call` without speaking.
- Wait patiently when someone says they're fetching the person.
- Never say "end_call" out loud or narrate that the call is ending.

## Inbound vs outbound

Same flow, same logic, same rules. Clone the outbound Intro and change the opening
sentence to reflect who called whom.

These are **Intro** openings, not the fallback sentences in the Voice Prompt
Configuration:

```text
Outbound: This is {bot_name} with {company_name} on a recorded line. I'm calling regarding...
Inbound:  Thank you for calling {company_name}. This is {bot_name} on a recorded line. How can I help you today?
```

On outbound, the disclosure lands here rather than in the configured first sentence,
which is typically just an identity check (`"Can I please speak to {name}?"`).

## References

| File | Contents |
| --- | --- |
| `references/architecture.md` | Component responsibilities, what prompting can't reach, prompt caching |
| `references/defaults.md` | Default config tables (call, audio, STT, TTS, LLM, turn detection) and default opening sentences |
| `references/global-rules.md` | Verbatim Voice Prompt rule blocks to copy |
| `references/ooo-and-callback.md` | Out of Office prompt and scheduled callback prompt templates |
| `references/examples/` | Real production scripts, indexed in its README |

Examples are real and in production, but predate parts of this guide — they may
contain conventions this document now supersedes. Follow this document; use the
examples for shape and phrasing.
