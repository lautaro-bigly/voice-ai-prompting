# Speech to Speech (S2S) Intros

The platform runs two voice engines, picked **per Intro** (`voice_engine` on
`tenant_ai_intros`): `cascade` (STT + LLM + TTS — everything else in this skill) and
`speech_to_speech`, where one model hears and speaks directly. S2S prompts are written
differently. This file is the whole S2S convention; the copy-and-fill templates are in
`s2s-templates.md`.

Use it when the user asks for S2S / speech to speech / GPT Live, or when the Intro being
read or edited has `effective_voice_engine: speech_to_speech`.

## How it works

| Part | Model | Job |
| --- | --- | --- |
| Voice model | `gpt-live-1` | Hears and speaks. Its prompt is the Intro body, because the Voice Prompt is only `{intro}` |
| Delegation model | `gpt-6-luna` | Woken when the voice model emits `<|SpawnThinking|>`. Reads the conversation, calls the tools, returns a short status |
| Tool audio | TTS outside the S2S engine | Speaks the tool's sentence. The tool row's default is pre-cached and plays instantly |

Flow: caller ↔ voice model → `<|SpawnThinking|>` → delegation model → tool with its
default values → cached sentence plays → the call ends, transfers, or a text goes out.
When nothing applies, the delegation model returns exactly `No backend action needed;
continue the conversation.` and the voice model carries on without reading it aloud.

**The voice model never calls a tool.** It decides *when* something should happen; the
delegation model decides *what* and does it. So every S2S Intro is two prompts that must
agree: the Intro body and its delegation prompt.

## Check every S2S prompt against OpenAI's guide

OpenAI's prompting guide for the voice model is the source for how these prompts are
written — the `Backchannel policy`, `Interruption policy` and `Delegation policy` labels
and the `<|SpawnThinking|>` token come from it and are kept exactly:

https://developers.openai.com/api/docs/guides/live-prompting
(`https://developers.openai.com/api/docs/guides/live-prompting.md` is the plain-text
version, with the starter template verbatim.)

**Fetch it on every S2S writing task** — a new Intro, a conversion, an edit to an Intro
body or a delegation prompt — before showing the user a preview. The page changes and
this file is a snapshot, so do not rely on memory of it. Then check the draft against
it:

- The three policy labels and the two `<|SpawnThinking|>` condition headings are present
  and spelled as the guide spells them.
- Backend work is requested with `<|SpawnThinking|>`, never with "delegate to the
  backend" or a spoken promise like "let me check that".
- Trigger conditions are concrete, on both the generate and the do-not-generate side.
- The Intro never states a price, a booking, a transfer or any action as done before the
  backend returns.
- The full task procedure is in the delegation prompt, not the voice prompt.
- Anything else the guide now recommends or warns against that the draft doesn't follow.

Report the result in one or two lines with the preview: what was checked, and anything
changed because of it. Two cases need more than that:

- **The guide contradicts this file or the templates** — tell the user what differs and
  ask which to follow; do not silently pick one. The Bigly rules in Non-negotiables
  (always the tool row's default, `{intro}`-only Voice Prompt, a delegation prompt per
  Intro, no appointment tool) are platform facts the guide knows nothing about, and they
  stand.
- **The page can't be fetched** — say so plainly, build from the templates, and note that
  the check was skipped.

## What changes from cascade

Everything in `SKILL.md` still applies unless this table overrides it.

| Cascade rule | On an S2S Intro |
| --- | --- |
| Tone, pace and interruption instructions are dead text | They work. The Intro carries a persona, a `# Voice and Energy` block, and backchannel and interruption policies |
| Voice Prompt holds persona, tools and the rule blocks; `{intro}` once under `# Introduction` | Its own Voice Prompt, **exactly** `{intro}` and nothing else, kept alongside the cascade one. Persona, rules, company info, callbacks and language all live in the Intro body |
| Variables injected once in the Voice Prompt's `# General Rule` | Injected in the Intro: `{bot_name}` and `{company_name}` in the first line, the rest in a closing `# Who you are calling` block |
| The persona never names the bot | The Intro opens with `You are {bot_name}, …` |
| `invoke the end_call tool. Parameters: -end_sentence:"…"` | The Intro never says "invoke", "delegate" or "say nothing". It says "end the call" / "transfer" / "send the link", and the Delegation policy says when to generate `<|SpawnThinking|>` |
| A custom `end_sentence` per ending; empty for voicemail, IVR, wrong number | **Always the tool row's default sentence.** Never empty, never custom, never translated, never with the caller's name |
| `transfer_number` and `transfer_sentence` written in the Intro's `# Transfer` | They are the `transfer_call` tool row's defaults, spelled out in the delegation prompt. The Intro's `# Transfer` only says when |
| Script lines are spoken as written | Quoted lines are points to make, in the model's own words. The only fixed words are the recording notice. Verbatim only if the user asks for it |
| The configured first sentence opens the call | It may not play — on inbound the caller speaks first. The Intro's `# Opening` carries the greeting and the recording notice |
| ALL CAPS branch keywords, `(You must receive…)` parentheticals | Plain conversational instructions: `Q1, find out: "{Q1}"` with the expected answer in a parenthetical |
| Recommended settings: LLM, TTS, STT, turn detection | Ignored by an S2S Intro. Its own settings are in `defaults.md`, Speech to Speech |

Unchanged: the recording notice on the first substantive turn; at least two questions;
`attribute_type`, `attribute_name` and a predefined `instruction` on every question; the
inbound side on every outbound Intro; E.164 numbers; callbacks inside `{working_hours}`
and never a time-zone question; no transfer before the last question; two pushes on a NO,
then the call ends; the ten blocking inputs; `read` → `preview_update` → `apply_update`.

## Non-negotiables

1. **Always the tool's default, spelled out.** `end_call` → the row's `default_sentence`;
   `transfer_call` → the row's `transfer_number` and `transfer_call_sentence`;
   `webhook_dispatcher` → its default id and sentence. Tool audio is TTS outside the S2S
   engine: the default is cached and plays instantly, anything else is generated on the
   fly and the call ends before it is spoken. In the delegation prompt every rule and
   every example writes the full call — `call end_call with end_sentence: "Thank you for
   your time!"` — never "use the default sentence". Read the real values from
   `tenant_ai_tools_configuration` first; never invent one.
2. **Every S2S Intro has its own delegation prompt.** It lives on the Intro, at
   `speech_to_speech_settings.reasoning_instructions`. A tenant-level default exists (the
   `speech_to_speech_settings` on `tenant_ai_voice_prompt`, which new Intros start from),
   but it is generic and knows nothing about this script — never leave an S2S Intro on it.
3. **The S2S Voice Prompt is only `{intro}`** — outbound and inbound both. It is a
   separate Voice Prompt; an existing cascade one is never overwritten to make it.
4. **Tool names stay in the Intro text** (`end_call`, `transfer_call`, `send_sms`,
   `webhook_dispatcher`), in the Delegation policy's `Backend tools:` list. A tool row is
   inert unless its name appears in the prompt, and on S2S the Intro is the prompt.
5. **Three places agree on every ending, transfer and text:** the Intro, the delegation
   prompt, and the tool row description. Change one, change all three. If the Intro
   spawns thinking on a condition the delegation prompt doesn't list, the backend answers
   "no action" and the caller hears a pause and nothing else.
6. **The bot sounds human** — a persona with a little backstory, small habits, real
   reactions, short turns. Professional unless the user says otherwise, never a script
   being read. See the `# Voice and Energy` block in `s2s-templates.md`.
7. **Converting a cascade Intro: create a new Intro.** The working one stays untouched as
   the fallback. See Converting a cascade Intro to S2S.

## Settings

Send `voice_engine: "speech_to_speech"` and the full `speech_to_speech_settings` object
on the Intro. The settings are stored per Intro and are not inherited at call time, so
send every key rather than trusting the tenant default. Values are in `defaults.md`,
Speech to Speech.

```json
{
  "voice_engine": "speech_to_speech",
  "speech_to_speech_settings": {
    "provider": "openai",
    "voice_model": "gpt-live-1",
    "voice": "marin",
    "reasoning_model": "gpt-6-luna",
    "reasoning_effort": "low",
    "reasoning_service_tier": "auto",
    "reasoning_verbosity": "medium",
    "reasoning_parallel_tool_calls": true,
    "reasoning_instructions": "<the delegation prompt>"
  }
}
```

On an existing S2S Intro, compare these against the defaults the same way as the
recommended settings in `SKILL.md`: list each difference and ask before changing it.

## The Voice Prompt

An S2S Intro runs under a Voice Prompt whose outbound and inbound sides are both exactly
`{intro}` — that string, nothing before or after it. A tenant can hold more than one
Voice Prompt, so the S2S one is its own prompt, **created next to the cascade one, never
in place of it.** A cascade Voice Prompt that is already there is never overwritten,
emptied or trimmed: the tenant's cascade Intros still depend on it.

Read `tenant_ai_voice_prompt` (`read`, then `list_prompts`) and take the first case that
fits:

1. **The active prompt is already `{intro}` on both sides** — nothing to do.
2. **A saved prompt is already `{intro}` on both sides** (usually named `S2S`) — reuse it.
   Do not create a second one; `save_prompt` always adds a new row.
3. **Neither** — create it with `save_prompt`: name `S2S`, `outbound_prompt` `{intro}`,
   `inbound_prompt` `{intro}`. Show the user what will be saved first. The active cascade
   prompt is left exactly as it is.

Then tell the user which Voice Prompt the S2S Intro has to run with, by name, and where
to pick it: **in the app, on the Intro itself — the Voice Prompt selector next to the
phone number and tag.** That selection is the user's step. The MCP can save a Voice
Prompt but cannot read or set which one an Intro uses (`tenant_ai_intros_update` rejects
any such field as not allowlisted), so never claim it was attached. It is also unknown
whether `tenant_ai_call_context_preview` follows the Intro's selection or only the
active prompt: if the preview shows the Intro wrapped in cascade text, say so and ask
the user to confirm the selection in the app rather than reporting the preview as clean.

Only when the tenant has no cascade Intro at all is it worth offering to make `{intro}`
the active prompt instead. Ask; never do it on your own.

## The delegation prompt

Written from the template in `s2s-templates.md`, set through `tenant_ai_intros_update` as
`speech_to_speech_settings.reasoning_instructions`.

- Hardcode everything: company, bot name, default sentences, numbers, webhook ids, SMS
  texts, start language. **No `{placeholders}`** — nothing is injected into it.
- Every ending, transfer and send condition in the Intro appears here, and the reverse.
- No empty `end_sentence` anywhere — not for voicemail, not for abuse.
- A `# No action` section returning exactly `No backend action needed; continue the
  conversation.`, listing the cases the voice model handles itself.
- Examples in every language the bot speaks, including the off-topic pair, the prank, and
  the language switch.
- Inbound and outbound share one delegation prompt, so direction-specific endings are
  labelled (`Outbound: voicemail…`, `Inbound: a sales call`).

**Appointments and callbacks have no tool.** When the caller agrees a day and time, the
delegation prompt calls `end_call` with the default sentence and nothing else. A layer
that runs after the call reads the transcript, tags it `MEETING_BOOKED` (or schedules the
callback) and sends an appointment to the calendar. So never write a booking tool into
the delegation prompt or the Intro. The `Meeting booking` / `Callback booking` line under
`Backend tools:` in the Intro describes that outcome so the voice model hands off at the
right moment; it is not a tool name. The day and time must be spoken and agreed in the
call, since the transcript is all that layer sees.

Any later edit that changes when the call ends, transfers or texts is two edits — the
Intro body and the delegation prompt — in the same preview.

## Tool rows

`tenant_ai_tools_configuration`: `end_call`, `transfer_call`, `send_sms`,
`webhook_dispatcher`. JSON shapes are in `s2s-templates.md`.

- Each speaking tool's `description` and parameter description say "always exactly the
  default". Rewrite any "pass an empty end_sentence for voicemail" wording.
- `transfer_call` needs `use_tool: true` and an E.164 `transfer_number`. The row key is
  `transfer_call_sentence`; the live tool schema exposes it as `transfer_sentence`.
- The default value *is* the cached audio. Do not change a default sentence unless the
  user asks.
- Rows are **tenant-wide**. A cascade Intro on the same tenant that passes custom goodbyes
  reads the same "always the default" description — tell the user before applying.
- `send_sms` text is not cached audio, so it may differ per language; the delegation
  prompt carries each exact text.

## What to settle with the user

On top of the ten blocking inputs in `SKILL.md`, and only where not already clear:

| Question | Why |
| --- | --- |
| Persona and behaviour — age, background, how playful | S2S follows tone directly. Professional by default, always human; humour is opt-in |
| Languages, and which one the call **starts** in | S2S switches mid-call, so one multilingual Intro replaces one per language |
| Script lines verbatim or reference? | Default is reference |
| Which Intro is the source, and the new title | Only when converting an existing Intro |
| Is this a demo line? | Old demo Intros fake the transfer with `end_call`; ask before making it real |

Transfer number and each disqualification are still read back before writing.

## Build order

1. Read: `tenant_ai_intros` (list, then the source Intro), `tenant_ai_tools_configuration`
   (every default sentence, number and description), `tenant_ai_company_variables`,
   `tenant_ai_voice_prompt`, and `tenant_ai_call_context_preview` on the source Intro.
2. Questions, as in cascade.
3. Outbound Intro body from the template; then the inbound body from the inbound
   differences table.
4. Check both bodies: no `invoke`, `delegate` or `say nothing`; `<|SpawnThinking|>`
   present; every tool name present; only valid placeholders.
5. Delegation prompt from the template. Then fetch OpenAI's guide and check both bodies
   and the delegation prompt against it (Check every S2S prompt against OpenAI's guide).
6. One `tenant_ai_intros_update` preview with bodies, opening sentences, questions,
   `voice_engine` and the full `speech_to_speech_settings`. No `id` creates the Intro.
7. Tool rows aligned to "always the default".
8. The `{intro}` Voice Prompt: reuse it or create it, per The Voice Prompt. Never
   overwrite a cascade one.
9. Callback Intro (tag `SCHEDULED_CALLBACK`) and, on transfer flows, the OOO Intro, where
   the flow needs them — written as S2S Intros too, each with its own delegation prompt.
10. Verify: read the Intro back and compare; run `tenant_ai_call_context_preview`
    outbound and inbound. `pipeline_config.engine.type` is `speech_to_speech`,
    `responses_options.instructions` is the delegation prompt, `tools` lists every tool,
    `tools_v2` shows the right `default_parameters`, and no literal `{…}` is left.
11. Hand over the test plan below.

## Converting a cascade Intro to S2S

"Take Intro A and do it as S2S" produces a **new** Intro B: A's script rewritten as an
S2S body, plus B's delegation prompt, engine and settings, in one write. A is never
edited — it stays as the fallback. Nothing is asked again that A already answers; only
the questions in What to settle with the user that A leaves open.

Read A in full, and read the Voice Prompt it runs under — on cascade half of A's script
lives there. Then map every piece:

| In the cascade source | In Intro B |
| --- | --- |
| Voice Prompt persona | First lines of the body (`You are {bot_name}, …`) and `# Voice and Energy`, expanded into a human persona |
| Voice Prompt rule blocks — Transfer Rule, Callback Rule, AI Assistant / Call Screener, AI Disclosure, User Questions, Language Rule, pricing | Rewritten into the body: `# Absolute Rules`, `# Callbacks`, `# Call Handling`, `# Voice and Energy`, `# Language` |
| `# Company Info` and `# General Rule` | `# Company Info` and `# Who you are calling` at the end of the body |
| `# Objective`, `# Initial Ask Handling`, `# Qualification Flow`, `# Rebuttal`, FAQs | Same flow, same order, same branches. Quoted lines become points to make; `Ask Q1: "{Q1}"` becomes `Q1, find out: "{Q1}"` |
| Every `invoke the end_call tool` | "end the call" in the body; the condition goes in `Generate <|SpawnThinking|> when` and in the delegation prompt's `# End call` |
| Each custom `end_sentence` | Dropped. If the line carried information (why they don't qualify), the bot says it as its own turn first, then the call ends on the default — and the delegation prompt says "after <Bot> explained…" |
| Empty `end_sentence` for voicemail, IVR, wrong number | The default sentence, like every other ending |
| `# Transfer` with `transfer_number` and `transfer_sentence` | `# Transfer` keeps only the condition. Number and sentence become the `transfer_call` row's defaults and are spelled out in the delegation prompt |
| `webhook_dispatcher` or `send_sms` invocations | A `Backend tools:` line, a SpawnThinking condition, and a section in the delegation prompt with the exact id, message or text |
| Dead text about tone, pace, noise | Tone and pace instructions are now live — keep the intent in `# Voice and Energy`. Noise and timing rules stay out |
| `questions`, `questions_inbound`, attributes, instructions | Copied unchanged |
| Opening sentences | Copied as fallbacks; the greeting and recording notice are written into `# Opening` |
| `tag`, `inbound_number` | The tag is copied. The number is not: B starts with none, and the user moves it when they switch over |

Things to check while mapping, and raise before writing:

- **Tool row defaults against A.** If A's transfer number or sentence differs from the
  `transfer_call` row, or A has several different goodbyes and the row has one default,
  say so. The row is tenant-wide; ask which value wins rather than changing it silently.
  No row for a tool A uses → create it, with the user's value read back.
- **A demo Intro that fakes its transfer with `end_call`** — ask before making it real.
- **Rules the cascade script was missing.** B gets the full template regardless: off-topic
  handling, call handling, transfer-did-not-connect, AI disclosure.
- **A's callback and OOO Intros.** Converting the main Intro leaves them on cascade, and
  they keep working there. B needs its own S2S callback Intro, and an S2S OOO Intro on a
  transfer flow — write them as new S2S Intros in the same pass, each with its own
  delegation prompt, and leave the cascade ones untouched.

Title B after A with the engine in the name (`<A's title> - S2S`) unless the user gives
one. Then follow the Build order from step 4.

## Multilingual Intros

- One Intro. Fix the start language; switch when the caller speaks another supported
  language in **whole sentences** or asks for it, and switch back the same way.
- A greeting alone is never language evidence — "bom dia" / "buen día", "hola" / "olá"
  sound alike on the phone. Stay in the start language until a full sentence.
- Similar languages are told apart by words, not accent ("você, não, tá" are Portuguese;
  "usted, bueno, dígame" are Spanish). Unsure → stay.
- Continue from the exact point; never comment on the switch, never end the call over it.
- Recording notice once per call, in the language being spoken.
- Configured questions are written in the start language; the Intro says to ask them
  naturally in the other. Answers store to the same attributes.
- Habits and example phrases are native to each language, never translated word for word.
- The `end_call` and transfer sentences stay the single cached default in every language.
  Only SMS text or a per-language transfer number changes, and the delegation prompt
  names each one and forbids mixing them.

## Test plan to hand over

Finish every S2S build or edit with what was written and this table, filled in:

| # | Say / do | Expected |
| --- | --- | --- |
| 1 | Pick up, confirm your name | Introduces itself, recording notice once, asks Q1 in the start language |
| 2 | Answer every question so you qualify | The transfer / text / booking fires, the cached default plays, no second goodbye |
| 3 | "Connect me" before the last question | Says it needs a couple of quick things first, keeps going |
| 4 | A disqualifying answer | Explains kindly, ends with the default sentence |
| 5 | "I'm busy, call me tomorrow at ten" | Callback booked, ends with the default sentence |
| 6 | "Take me off your list" | Ends with the default sentence |
| 7 | "Do you do roofs?", then ask again | First a redirect, second time the call ends |
| 8 | Insult the bot; swear about your day | As the script says for each — the two are different cases |
| 9 | Speak the other language mid-call | Switches, continues from the same point, end sentence unchanged |
| 10 | "Are you a robot?" | Says yes, it's an AI assistant, returns to the question |
| 11 | Listen | Varied confirmations, small habits, short turns, no read-backs |

If the bot pauses and nothing happens, the delegation model answered "no action": the
delegation prompt is missing that condition. Fix the Intro and the delegation prompt
together.

## Gotchas

- `<|SpawnThinking|>` may be stored HTML-escaped (`&lt;|SpawnThinking|&gt;`). Calls work
  either way; unescape before comparing a read-back.
- If the preview shows the Intro wrapped in other text, the call is still assembling
  under the cascade Voice Prompt — the `{intro}` one has not been selected yet.
- A new Intro starts from the tenant's S2S defaults, including the generic
  `reasoning_instructions`. Creating an S2S Intro without sending its own is the silent
  failure this file exists to prevent.
- `{FirstName}` is not a placeholder and renders blank; the contact's name is `{name}`.
- The platform appends `(wait for user to speak) [ instruction ]` inside an injected
  `{Q#}`. Expected.
- The preview's second message can show a blank name. Preview artifact.
- A new Intro has no `inbound_number`; the user assigns it.
- Absolute Rules against an inbound direct transfer (existing customer, has an
  appointment, asks for someone by name): write the exception inside the rule itself, or
  precedence blocks the transfer.
- Older S2S prompts say "invoke end_call", "Delegate to the backend when", and pass an
  empty `end_sentence` for voicemail or abuse. All three are obsolete — replace them when
  the Intro is touched.
- If the offer itself is roofing or windows, swap the off-topic examples.
