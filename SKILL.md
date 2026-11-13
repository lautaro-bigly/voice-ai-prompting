---
name: voice-ai-prompting
description: Write, review, and debug prompts for the Bigly Sales Voice AI platform — outbound and inbound phone scripts that qualify leads, transfer to a live agent, or schedule appointments, on either voice engine (cascade STT-LLM-TTS, or speech to speech / S2S / GPT Live). Use when creating or editing a Voice Prompt, an Intro, an Out of Office prompt, or a scheduled callback prompt; when writing an S2S Intro and its delegation prompt, or converting a cascade Intro to S2S; when asked why a voice bot transferred early, skipped a question, or ignored a timing/tone instruction; or when reviewing an existing call script against platform conventions.
---

# Voice AI Prompting

Prompts for this platform are phone call scripts. They drive a live conversation
with tool calls (`transfer_call`, `end_call`) as the terminal actions, so the cost
of a loose instruction is a real call transferred to the wrong place or dropped on
a real person.

## Two engines: cascade and speech to speech

Every Intro runs on one of two voice engines, picked per Intro (`voice_engine`):

- **Cascade** — STT + LLM + TTS. Everything in this document describes cascade.
- **Speech to speech (S2S)** — one model (GPT Live) hears and speaks, and hands tool
  calls to a second model through a **delegation prompt**. Its prompts are written
  differently, and several rules below are reversed for it.

**Before writing or editing anything for an S2S Intro, read
`references/speech-to-speech.md` in full**, and build from `references/s2s-templates.md`.
It is S2S when the user says S2S / speech to speech / GPT Live, or when the Intro in
question reads back `effective_voice_engine: speech_to_speech`. A new build that names no
engine and has no source Intro to follow: ask once which engine. Never apply cascade
conventions to an S2S Intro, or the reverse. The short version:

- Every S2S Intro **always** has its own delegation prompt, in its
  `speech_to_speech_settings.reasoning_instructions`. The tenant default is never enough.
- The S2S Voice Prompt is exactly `{intro}` and nothing else. If the tenant has none,
  create it as a **new** Voice Prompt — never overwrite a cascade one that is already
  there. Persona, rules and company info all live in the Intro body.
- Callback and OOO Intros that an S2S flow needs are written as S2S Intros too.
- The Intro never invokes a tool; it generates `<|SpawnThinking|>` and the delegation
  model makes the call — always with the tool row's default `end_sentence`,
  `transfer_number` and `transfer_sentence`, never a custom or empty one.
- Tone, pace and interruption instructions work on S2S. They are not dead text there.
- "Take Intro A and do it as S2S" creates a **new** Intro with the rewritten body and its
  delegation prompt, and leaves A untouched.
- S2S settings are in `references/defaults.md`, Speech to Speech.

## Know what the prompt can actually control

This section is about cascade. The platform is four independent components. The prompt is the LLM (GPT-4.1 Mini)
and **only** the LLM. Instructions aimed at the other three silently do nothing:

| You want | Owned by | Prompt can change it? |
| --- | --- | --- |
| Conversation flow, questions, business logic, wording | LLM | Yes |
| When the bot decides the caller stopped talking, response delay | Turn detection (LiveKit) | No |
| Ignoring background noise, transcription accuracy | STT (Parakeet) | No |
| Voice, tone, pacing, speaking softly/slowly | TTS (Fish Audio) | No |
| How a given word comes out | TTS (Fish Audio) | Only by respelling it |

So "wait two seconds before answering", "ignore background noise", and "speak in a
soft, slow voice" are dead text. Fix those in the component config, not the prompt.

**On an existing prompt, delete dead text on sight.** This is a requirement, not a
suggestion. Whenever the skill reads, reviews, or edits a prompt that is already on the
tenant and finds an instruction the LLM cannot act on — a timing rule, a noise rule, a
voice or pacing rule, anything owned by turn detection, STT, or TTS — remove it as part
of the change, and say what was removed and why. Do not leave it in because it is
harmless: every dead line is clutter the model has to read past on every call. If the
instruction had a real intent behind it, carry that intent to the component config
(`references/defaults.md`) or, where possible, rewrite it as a text rule per the next
paragraph.

**The boundary is over the component, not its input or output.** The prompt can't
change how TTS renders a string, but it fully controls *which* string TTS receives —
so pronunciation is reachable by respelling. `"Never output digits; write numbers as
words"` is a legitimate prompt rule, because "fifty" and "50" are different input.
Likewise the prompt can't improve transcription accuracy, but it can tell the LLM how
much to trust a transcript: STT mangles proper nouns, so `"do not trust names in the
transcript; branch on intent only"` works where `"transcribe names accurately"` is
dead text. Rewrite an unreachable instruction as one about text the LLM produces or
consumes. See `references/architecture.md`.

## From a request to a script

This skill is normally used alongside a connected Bigly platform MCP — one connection
per tenant. The job is to turn a plain request ("we want to call our aged leads and
transfer the interested ones") into the full set of prompts and config that the
platform needs, and write them through that connection.

**Ask only for what you cannot derive.** These ten things have no default, and a
script is not buildable without them:

1. Company name, and 2–3 sentences on what the business does (becomes `# Company Info`).
2. Bot name.
3. Goal — qualify → transfer, qualify → schedule, or both.
4. The qualification questions, with the answer each must produce.
5. What disqualifies someone, and what the bot says when they don't qualify.
6. Transfer destination number — transfer flows only.
7. Business hours and time zone — transfer flows only. Working hours exist so the bot
   knows when a live agent can pick up. With no transfers there is no one to pick up,
   so do not ask for them and never set them.
8. Outbound, or inbound-only. An outbound script always gets its inbound side as well —
   see Inbound vs outbound — so the only question is whether outbound is in scope.
9. Language, if not English-only.
10. When an AI call assistant answers for the contact: **handle it** or **hang up
    immediately** (`references/global-rules.md`, AI Assistant / Call Screener). A
    business decision; handle is the usual answer for transfer flows.

**Every one of these is blocking.** If the request leaves one out, ask for it before
building anything. Do not guess it, do not stand in a placeholder such as
`+1XXXXXXXXXX` or `[transfer number here]`, and do not quietly build the script without
the part that needs it. No transfer number on a transfer flow means you ask for the
transfer number — not that you write an Intro with no `# Transfer` section. Ask for all
the missing items in one message rather than one at a time.

**Derive everything else — don't ask.** The value of every setting in
`references/defaults.md` (you still ask whether to *apply* them — see Apply the
recommended settings);
every rule block in `references/global-rules.md` (but which AI Assistant / Call
Screener setup to use is item 10 above — ask it); the section layout; the default
pricing rule; E.164 reformatting of any number given loosely (`707-555-0123` →
`+17075550123`); and the fact that an OOO prompt and a callback Intro are required at
all. Asking about these wastes the user's turn on a decision the skill already made.

**Confirm before writing, never assume:** the transfer number read back digit by
digit, and each disqualification branch. A wrong transfer number routes live callers
to a stranger, and a wrong disqualification hangs up on good leads — neither is
recoverable after the fact.

**Read before you write.** On an existing account, `tenant_ai_voice_prompt` (read),
`tenant_ai_intros` (list), and `tenant_ai_company_variables` (read) tell you what is
already there — including variables sitting empty and Intros you'd otherwise duplicate.

**Then build, in this order.** Each step depends on the one before it:

1. Questions first, each with `attribute_type`, `attribute_name`, and a predefined
   expected output (`instruction`) — the flow's branches are written against them.
2. Outbound Intro — objective, pickup handling, qualification flow, transfer or
   scheduling, rebuttals.
3. Voice Prompt — persona, tools, the standard rule blocks, and closing with
   `# Introduction`, the Callback Rule, and the `# General Rule` block, in that order.
4. Inbound side of the same Intro — required, never "if in scope". Fill
   `content_inbound`, `first_inbound_sentence`, and `questions_inbound` on the Intro from
   step 2, changing only who called whom.
5. Callback Intro — clone step 2, acknowledge the requested callback. Tag
   `SCHEDULED_CALLBACK`.
6. OOO prompt — transfer flows only. Tag `after_office_hours`. The main inbound
   flow, ending in an appointment instead of a transfer. Tell the user they must
   create and pick the AI meetings calendar themselves (`references/ooo-and-callback.md`).
7. Config, per the next section.
8. `tenant_ai_call_context_preview` — assemble the real prompt without dialing, and
   confirm every `{variable}` resolved instead of reaching the LLM as literal braces.

**Every write is `read` → `preview_update` → `apply_update` with a per-tool
confirmation string.** Show the preview diff before applying: this is a live tenant, and
an applied change affects calls placed minutes later. On `tenant_ai_intros`,
`apply_update` without an `id` creates a new Intro instead of updating one. Details and
the confirmation strings are in `references/platform-mcp.md`.

Run the hard-requirements list below against the result before calling it done. If the
MCP is unavailable, produce the same artifacts as text for the user to paste — the
deliverable is identical either way.

## Apply the recommended settings

The script is only half the deliverable. The component settings in
`references/defaults.md` apply to every prompt and are never set automatically. Call
them the **recommended settings** when talking to the user — that is what they are.

This step is never skipped. A script delivered without it runs on whatever the tenant
happened to have, which is the most common way a finished prompt ends up misconfigured.
On every prompt task, new or existing, read the current config with
`tenant_ai_voice_prompt` action `read` and compare it against `references/defaults.md`,
which lists the MCP field name for each setting. (`tenant_ai_settings` is *not* this —
it holds tenant plumbing like SIP and DNC thresholds.) Then always tell the user where
the config stands:

- **Everything matches** — say in one line that the recommended settings are already
  in place, and move on. Nothing to write.
- **Something differs** — never overwrite it silently, and never leave it as it is
  without asking. A different value may be deliberate. List each difference (setting,
  current value, recommended value) and ask whether to apply the recommended settings.
  Apply only what the user approves — they may want to keep one override and fix the
  rest. If they decline, drop it and don't ask again for that prompt.

`Maximum Call Duration` is a judgment call rather than a fixed value; count it as
matching whenever it clears the 4-minute floor.

After applying, report which values changed. If the MCP can't read or write the
config, say so plainly and output the filled-in tables for the user to enter — never
finish a prompt with the recommended settings silently unapplied.

## Check what's missing

On every task that touches an existing Voice Prompt or Intro, check the part you're
touching for the required rules while you read it. Look for what each block **does**,
not its heading — `#Callback Rule`, `Calling back instructions` and `[CALLBACK]` are all
the Callback Rule if they do its job:

| Required | Present if something in the prompt… |
| --- | --- |
| Callback Rule | books a busy caller a time within working hours, then `end_call` |
| AI Assistant / Call Screener | says what to do when a screening assistant answers |
| Transfer Rule | blocks a transfer before every question is answered, and on "put me through" — transfer flows |
| General Rule | injects the company variables, `{name}` and `{date}` in one block at the end |
| `{intro}` | appears exactly once |
| Tools | lists `transfer_call` — transfer flows |
| OOO and callback Intros | exist, tagged `after_office_hours` and `SCHEDULED_CALLBACK` — transfer flows |

- **Missing** — suggest adding the standard block from `references/global-rules.md`.
- **Present under another name or wording** — not missing. If it differs from the
  standard block, suggest replacing it and say what differs.
- **Never add or replace without approval.** These are suggestions, like the
  recommended settings.

This is not a full audit. Wording, conventions and leftover rules are reviewed only when
the user asks to audit the tenant, or on a tenant's first onboarding: then run the whole
Hard requirements list and the recommended settings against every prompt and Intro.

**Remind about the audit** once per conversation, at the end of the first task that
wrote a change: `Tip: run a full audit of this tenant once in a while — ask me to "audit
this tenant".` If the check above found something missing, say so in the same line. Not
on read-only questions, and not twice.

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

The sharpest case of this split is `transfer_call`. The Voice Prompt *lists* the tool
and *guards* it with the Transfer Rule — it never *invokes* it. Every invocation, with
its `transfer_number` and `transfer_sentence`, lives in an Intro's `# Transfer` section:
the destination and the moment to transfer belong to one conversation, while the Voice
Prompt is shared by every Intro on the tenant, including the after-hours one — a Voice
Prompt that transfers on "can I talk to a human" sends an after-hours caller to a
closed call center. `end_call` is the contrast: global rules such as the Callback Rule
and Language Rule invoke it from the Voice Prompt, and that is correct.

## Hard requirements

Check every script against these. They are not stylistic.

- The exact phrase **"on a recorded line."** is spoken on the bot's first
  substantive turn, on every flow — outbound, inbound, callback, and Out of Office.
  Note this is usually *not* the configured First Outbound Sentence: that field is a
  fallback the Intro overrides (see `references/defaults.md`), so an outbound script
  opening with a bare `"Can I please speak to {name}?"` carries the disclosure on the
  next turn, in the Intro's pitch. Inbound and OOO greet and disclose in one sentence.
  This is a requirement on the **author**, satisfied by hardcoding the phrase into
  that opening text. It is never written as a rule for the LLM — a `# Recording
  Notice` block telling the model to "make sure to say it in your first turn" is
  dead weight, because the phrase is already in the sentence the model reads out.
  If a review finds such a rule, delete it and check the opening text instead.
- At least **two questions**, even when one would do. This is a guard against the
  bot racing to a transfer on a single answer. The OOO Intro carries the main
  Intro's questions, so it is no exemption.
- Every phone number is full U.S. E.164 with the `+1` prefix: `+17075550123`.
- Every tool the flow needs is listed in the Voice Prompt. A transfer flow without
  `transfer_call` listed fails silently.
- The Voice Prompt carries one of the two `# AI Assistant / Call Screener` setups from
  `references/global-rules.md`, verbatim. The `end_call` tool description is left as
  deployed; the block alone decides the behaviour.
- `transfer_call` is listed in the Voice Prompt and invoked only in Intros. A Voice
  Prompt that invokes it is wrong whatever the number — see The two layers.
- Every `end_call` that says goodbye passes the line as its `end_sentence` parameter.
  A `say: "..."` followed by a bare tool call plays the tenant default sentence after
  it — two goodbyes. See Writing conventions.
- Every `transfer_call` invocation is the `# Transfer` form from Writing conventions:
  invoke, the no-message-in-this-turn instruction, then `transfer_number` and
  `transfer_sentence` as parameters — never `Say the transfer_sentence, then invoke`,
  which plays the sentence twice.
- The NO path of `# Initial Ask Handling After Opening Sentence` is the two-attempt
  push rule from Standard Intro structure, selling script or not, with its final
  `proceed to` pointing at the section that actually follows.
- The YES path goes directly to Q1. No script asks permission to ask questions
  (`"Is it okay if I ask you a few questions?"`) — a transition sentence is allowed, a
  question that waits for consent is not. See Standard Intro structure.
- Every question asks for one thing. No option lists, no appended reasons, no two
  questions in one sentence — callers answer the first half and talk over the rest.
  See Writing conventions.
- Scheduling scripts must **not** list `transfer_call` at all, and must state
  explicitly that transfers are not available. `prompt_type` must be
  `meeting_scheduler`, not `transfer_call` — the enum has to agree with the script.
- Company variables define at minimum `bot_name`, `what_is_your_goal`, `company_name` —
  and none of the ones the script relies on may be left empty. See
  `references/variables.md`.
- The Voice Prompt ends with the Callback Rule and then the `# General Rule` block
  from `references/global-rules.md`, verbatim. No company variable, `{name}`, or
  `{date}` appears in the Voice Prompt outside those two blocks — the opening
  sentence of a flow is the only other place one may be spoken. See Writing
  conventions.
- Neither the persona nor the `who_you_are` variable names the bot — no `{bot_name}`
  and no hardcoded name. Role and character only. See Writing conventions.
- `{intro}` appears in the Voice Prompt **exactly once**, on its own line under an
  `# Introduction` heading placed directly above the Callback Rule. Zero is a Voice
  Prompt with no script; two or more pastes the entire Intro body into the prompt
  again at every occurrence. See Writing conventions.
- No instruction aimed at turn detection, STT, or TTS survives in any prompt the skill
  touches — new or existing. "Ignore background noise", "wait two seconds", "speak
  slowly" and their kin are removed on sight, not preserved. See Know what the prompt
  can actually control.
- Default pricing rule, unless the use case overrides it:
  `You can't discuss the price, only a licensed agent can do so.`
- Working hours and an Out of Office Intro (tagged `after_office_hours`) exist on
  transfer flows, and only there. Both describe when live agents answer the phone. A
  script with no transfers — scheduling, information, anything else — has no live
  agents, so it has no working hours and no OOO Intro. Never set working hours on it.
- The OOO Intro never has an `inbound_number`. A bound number serves the OOO prompt on
  every call to it, in working hours too. If one is set, clear it without asking. See
  `references/ooo-and-callback.md`.
- The OOO Intro qualifies like the main one and offers an appointment, never a
  callback, and never says the office is closed. It needs an AI meetings calendar the
  user creates and picks themselves — the skill can't. `prompt_type` stays
  `transfer_call` (unverified, see `GAPS.md`).
- A scheduled callback Intro exists, tagged `SCHEDULED_CALLBACK`.
- Every outbound Intro — main script and callback alike — has its inbound side filled:
  `content_inbound`, `first_inbound_sentence`, `questions_inbound`. The OOO Intro is the
  one exemption, because it is inbound-only and has no outbound side to mirror. See
  Inbound vs outbound.
- Every question — outbound and inbound, on every Intro — has an `attribute_type`, an
  `attribute_name`, and an expected output (`instruction`) that is one of the platform's
  predefined values, never free text. All three are mandatory, with no exceptions: a
  question missing any of them is not finished. The values are listed in
  `references/platform-mcp.md`. **Demos** are the one variation: when the user
  explicitly says they are building a demo — never inferred from fake data or a
  placeholder company — confirm it with them first, then set every expected output to
  `must get an answer.`. `attribute_type` and `attribute_name` stay mandatory.

## Writing conventions

**Sections.** Markdown headings, referenced by name in the flow (`proceed to
#Qualification Flow`). Sections make the prompt modular — a block can be reordered
or swapped without touching the rest.

**Variables.** `{}` is a real runtime injection: `{name}`, `{Q1}`, `{date}`,
`{working_hours}`. `[]` is a reference for the LLM, not an injection:
`"Hello [user first name]"` tells the model to look up a value it already has. For
the standard set and which layer injects each, see `references/variables.md`.

**Inject each variable once, in the `# General Rule` block.** Questions used once
(`{Q1}`) are injected inline in the Intro. Every other variable — the company
variables, the contact's `{name}`, `{date}` — is injected exactly once, in a
`# General Rule` block that is the **last thing in the Voice Prompt**, and referenced
with `[]` anywhere else. The verbatim block is in `references/global-rules.md`. The
Callback Rule is the one other block allowed to carry variables (`{working_hours}`,
`{date}`), and for the same reason it sits immediately before `# General Rule`. The
`# Introduction` block holding `{intro}` sits just above those two. So the tail of
every Voice Prompt is, in order: `# Introduction`, Callback Rule, `# General Rule`.
Everything above that tail is static text, which is what keeps the prompt's cacheable
prefix long — see `references/architecture.md`.

The opening sentence of a flow is the one place a variable may be spoken outside
that block: `This is {bot_name} with {company_name} on a recorded line` and
`Can I please speak to {name}?` are both correct there. A second `{bot_name}` or
`{company_name}` anywhere else — the persona, a rule, a rebuttal — is a mistake:
it fragments the cache and repeats information the model already has.

**`{intro}` is injected once and never referenced.** It is not a variable holding a
name or a date — it is the whole Intro body, substituted in full at every place the
token appears. One occurrence is the join between the two layers; a second one
duplicates the entire script inside the prompt. So the Voice Prompt has exactly one
`{intro}`, as a standalone block near the end of the prompt — directly above the
Callback Rule and `# General Rule`, since it varies per Intro and belongs in the
dynamic tail with them:

```text
# Introduction
{intro}
```

Never write `{intro}` inside a sentence. Rules that need to point at the script point
at its sections by name (`go back to the question you left in #Qualification Flow`,
`proceed to #Transfer`), never at `{intro}`:

```text
WRONG:
For the qualification questions, follow {intro}.
Rebuttals are handled as described in {intro}.

RIGHT:
Rebuttals are handled in #Rebuttal.
```

**The persona never names the bot.** Not as `{bot_name}`, and not hardcoded. The
name is already in the prompt through the opening sentence, so naming it again adds
nothing and ties the persona to one campaign — a persona is meant to be reusable
across scripts and bot names. This applies equally to the `# Persona` block and to
the `who_you_are` company variable. Describe the role and character only:

```text
RIGHT:
You are a man in his early thirties working as a mortgage coordinator. You are confident, direct, friendly and professional. You keep the call brisk and businesslike without ever sounding cold or rushed. You sound like a real person, never like a script. If you have to cover the same ground twice, rephrase it a new way - never repeat the same exact wording twice, even if the caller cut you off the first time.

WRONG:
You are {bot_name}, a man in his early thirties working as a mortgage coordinator. ...

WRONG:
You are Tyler, a man in his early thirties working as a mortgage coordinator. ...
```

Same for `who_you_are`: `You are a sales assistant`, never `You are Juan, a sales
assistant`.

**One question, one part.** Every question the bot asks — configured `{Qn}` text and
inline asks alike — asks for exactly one thing and stops. No lists of options, no
justification tacked on, no second question in the same breath. The reason is the
medium: callers start answering as soon as they hear enough to answer, so on a
two-part question they talk over the second half. TTS keeps playing text nobody is
listening to, the turn gets tangled, and the caller feels interrupted by the bot.

```text
WRONG:
"What are you looking for — windows, doors, or something else?"
"What is your zip code? It's to help you better, and please spell it out."
"What's your house address, do you live in Florida?"

RIGHT:
"Are you looking for windows?"
"What's your zip code?"
"What's your address?"
```

If a question genuinely needs two facts, it is two questions. If it needs options,
ask a yes/no about the most likely one and let the flow branch. Keep every question as
short as it can be and still be answered.

**State the expected answer** in parentheses after every question. This is how the
model knows it has collected enough to move on. The question's `instruction` field — its
expected output — is set as well, always to one of the predefined values in
`references/platform-mcp.md`. Those values are generic, so anything more specific (map
the answer to an enum, a date format) goes in the body parenthetical, never in
`instruction`. Set both.

```text
Ask Q1: "{Q1}"
(You must receive a YES/NO answer.)

If YES or EQUIVALENT INTENT, proceed to Q2.
If NO or EQUIVALENT INTENT, invoke the end_call tool. Parameters:
-end_sentence:"Totally understand. If anything changes, we're here to help. Have a great rest of your day."

Ask Q2: "{Q2}"
(You must receive a valid U.S. state.)
```

**Ending a call.** The closing line goes in `end_call`'s `end_sentence` parameter, as
in the example above — the platform speaks it and then hangs up. Never deliver it as a
spoken message followed by the tool call:

```text
WRONG:
If NO or EQUIVALENT INTENT, say: "No problem at all. Thanks for calling and have a great day." Then invoke the end_call tool.

RIGHT:
If NO or EQUIVALENT INTENT, invoke the end_call tool. Parameters:
-end_sentence:"No problem at all. Thanks for calling and have a great day."
```

The wrong form still ends the call, but with no sentence specified the tool falls back
to the tenant's `default_sentence`, so the caller hears the script's goodbye and then
the generic default one. Putting the line in `end_sentence` makes it the last thing
said and keeps it under the script's control. Silent endings (voicemail, IVR, wrong
number) are the opposite case: invoke `end_call` with an empty `end_sentence` and no
message, rather than leaving it unspecified and letting the default play. See
`references/platform-mcp.md`.

**Transferring a call.** Same principle: `transfer_number` and `transfer_sentence` are
parameters of `transfer_call`, so the Intro's `# Transfer` section is the invocation,
an explicit instruction to output nothing else in that turn, and the parameters:

```text
# Transfer

Invoke the transfer_call tool. Do not output any assistant message in the same turn as the transfer_call tool. The only user-facing message should be the tool's transfer_sentence. Parameters:
transfer_number: +17075550123
transfer_sentence: "Great. I'm going to place you into our loan specialists' call queue. They should answer within a minute."
```

The instruction is not decoration. The tool speaks `transfer_sentence` itself, so any
assistant message in the same turn is heard on top of it — typically the same sentence
twice. That is exactly what a list of values followed by `Say the transfer_sentence,
then invoke the transfer_call tool with transfer_number` produces. And no restated
timing guard (`Do not transfer before Q1 has been answered`): the Voice Prompt's
Transfer Rule already owns that.

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

**Initial Ask Handling** branches on the caller's answer to the pitch. YES proceeds to
the next section — straight into Q1, never through a permission question. The pitch
was the ask; a YES to it is the consent. Writing `Ask: "Is it okay if I ask you a
couple of questions?"` between the pitch and Q1 adds a turn, invites a second NO, and
makes the bot sound unsure of itself. A one-line transition that is not a question is
fine (`Say: "Great, let me see if you qualify." then immediately proceed to Q1`);
anything that waits for an answer is not. NO is this push rule, on essentially every script — selling or not,
the point is to keep the caller in the conversation long enough to qualify them or get
the message across. It has proven itself in production:

```text
If NO or EQUIVALENT INTENT (not interested) gently push for a sale for a maximum of two times. If the caller still refuses after two attempts then invoke the end_call tool. Parameters:
-end_sentence:"Thank you for your time, call us back if you change your mind!". If the caller accepts then immediately proceed to #Qualification Flow.
```

Point the final `proceed to` at whatever actually follows in that script —
`#Qualification Flow`, `#Conversation`, `Q1` — and keep the cap: two attempts, then
the call ends. On a script that isn't selling, `for a sale` becomes what the script
actually wants, such as `to hear the message`. Only a use case that explicitly wants
NO taken at face value goes without the push.

**Rebuttal** handles objections that block the goal: why am I being called, general
hesitancy, wanting information before answering, skepticism. Acknowledge, answer
briefly, steer back to the flow. Never argumentative, never more persistent than
the script allows.

## Standard Voice Prompt rule blocks

Copy these into the Voice Prompt of essentially every script. Verbatim text is in
`references/global-rules.md`.

- **Transfer Rule (Priority)** — never transfer before every qualification question
  is answered, and never because the caller says "hold" / "connect" / "put me
  through." A guard on the tool, not an invocation of it — see The two layers.
- **AI Assistant / Call Screener** — what the bot does when a call-screening assistant
  answers for the contact. Two setups, handle or hang up immediately; one of them is
  always present. The `end_call` tool description is not touched.
- **AI Disclosure (Priority)** — asked whether it is a recording, a robot or an AI, the
  bot says it is an AI assistant for the company, over any scripted identity answer.
- **Callback Rule** — caller is busy, book a time within `{working_hours}`, never
  in the past, then `end_call`. Never ask the caller's time zone — the backend
  resolves it when it books the callback, so a script that asks is wrong.
- **User Questions** — answer, then return to the exact question you left. Never
  skip a question without an answer.
- **Language Rule** — English by default unless the use case says otherwise.
- **Company Info** — 2–3 paragraphs for answering questions about the business.
  Include it even if you only have placeholder text to replace later.
- **General Rule** — the single block where every variable is injected. Always the
  last block in the Voice Prompt, with the Callback Rule directly above it and
  `# Introduction` above that.

## Edge cases worth handling explicitly

- Complete all questions before any transfer or scheduling.
- Distinguish a human gatekeeper / AI assistant from voicemail — they need
  different handling. Voicemail and IVR should `end_call` without speaking — an
  empty `end_sentence` and no message, since an unspecified one plays the tenant
  default.
- Wait patiently when someone says they're fetching the person.
- Never say "end_call" out loud or narrate that the call is ending.

## Inbound vs outbound

**Every outbound Intro has an inbound counterpart. Always.** People call back the
number that just called them — a missed call, a voicemail notification, a callback they
asked for — and that return call selects the same Intro on its inbound side. An Intro
with an empty inbound side answers that caller with no script. This is not a scope
question for the user; it is part of building an outbound Intro, so never ask whether
they want it and never deliver an outbound Intro without it. The one Intro exempt is
the Out of Office one, which is inbound-only by design (`references/ooo-and-callback.md`).

Same flow, same logic, same rules. Fill the inbound fields on the same Intro from the
outbound ones and change the opening sentence to reflect who called whom.

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
| `references/speech-to-speech.md` | S2S Intros: how the engine works, what changes from cascade, the delegation prompt, converting a cascade Intro, test plan |
| `references/s2s-templates.md` | S2S Intro body, `# Voice and Energy` block, inbound and callback differences, delegation prompt, tool rows |
| `references/platform-mcp.md` | The MCP tool surface, the write handshake, Intro tags, question fields |
| `references/architecture.md` | Component responsibilities, what prompting can't reach, prompt caching |
| `references/defaults.md` | Default config tables (call, audio, STT, TTS, LLM, turn detection) and default opening sentences |
| `references/global-rules.md` | Verbatim Voice Prompt rule blocks to copy |
| `references/variables.md` | The standard `{}` variable set, and which layer injects each |
| `references/ooo-and-callback.md` | Out of Office prompt and scheduled callback prompt templates |
| `references/examples/` | Real production scripts, indexed in its README |

Examples are real and in production, but predate parts of this guide — they may
contain conventions this document now supersedes. Follow this document; use the
examples for shape and phrasing.
