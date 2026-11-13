# Coverage Gaps

What this skill does **not** currently document. Compiled from reading the whole skill
against the live platform surface (`tenant_ai_*` MCP tools on a real tenant) and against
`references/examples/01`.

Each item is marked with what kind of gap it is:

- **[undocumented]** — confirmed to exist on the platform or in a production script; the
  skill simply doesn't mention it.
- **[unknown]** — nobody has written down how it works and I could not determine it from
  the outside. Needs someone with platform knowledge, or more MCP reads.
- **[decision]** — not a missing fact but an unmade scoping call.

Nothing here is a blocker for writing a normal transfer or scheduling script. The items
marked **high** are the ones that cause silent breakage or wasted rework.

---

## High impact

**Prompt versioning and rollback** — [undocumented] · `tenant_ai_voice_prompt` has
`list_prompts`, `save_prompt`, `delete_prompt` and returns a `prompt_library` array. This
is effectively version control for prompts, and a skill whose whole job is editing live
prompts says nothing about snapshotting before a change or rolling one back. Highest-value
gap on this list: it's the safety net for everything else the skill does.

**Same-turn tool constraints** — [undocumented] · Production prompts carry rules like
`"Do not invoke end_call in the same turn as webhook_dispatcher"`. These read as
platform-level execution constraints, not stylistic preferences. The `transfer_call`
counterpart — no assistant message in the same turn as the tool — is now covered by the
"Transferring a call" convention in `SKILL.md`; the webhook one is still learnable only
by reading example 01.

**Custom output cannot see the prompt** — [undocumented] · A question's `instruction`
runs against the transcription only — it has no access to the Intro or Voice Prompt.
This is why normalization rules have to restate their enum inline rather than referencing
a section. Noted as a parenthetical in example 01 and nowhere else. It silently produces
wrong stored answers when violated.

**Hidden questions** — [unknown] · Example 01 uses `hidden_question` and
`hidden_lang_question` to store computed values without injecting them into the Intro. The
`questions` array schema I inspected (`question`, `instruction`, `attribute_name`,
`attribute_type`) has no visible "hidden" field, so *how* this is configured is
unresolved. The technique matters — it's how you persist a calculation or a
language-detection result.

**Where `{working_hours}` comes from** — [unknown] · It is not a company variable, but
the Callback Rule and the OOO body both depend on it. `tenant_ai_followup` accepts
"scalars (totals + working hours)", which is the most likely source and worth one read to
confirm. If it resolves to nothing, the Callback Rule asks the caller to pick a time from
an empty list.

**Transfer failure paths** — [unknown] · Nothing covers what happens when a transfer
fails, no agent picks up, or the destination is busy. The existence of a
`TRANSFERRED_DROPPED` intro tag implies the platform has a defined path for at least one
of these. Undocumented, and it's the failure mode with a real person on the line.

**Attaching a Voice Prompt to an Intro** — [unknown] · A tenant holds several Voice
Prompts, and an S2S Intro runs under its own `{intro}`-only one next to the cascade
prompt. The MCP can create it (`save_prompt`) but shows no field that selects a saved
prompt for an Intro or campaign, so `speech-to-speech.md` hands that step to the user.
If the MCP gains the field, the skill should set it and verify with the call-context
preview.

**OOO on S2S** — [unverified] · `speech-to-speech.md` writes the OOO Intro of an S2S flow
as S2S with its own delegation prompt. No production OOO Intro on S2S has been checked.

**No S2S example** — [decision] · `references/examples/` has no S2S script. The templates
in `s2s-templates.md` came from production builds, but an anonymized Intro plus its
delegation prompt would show how the two line up.

---

## Undocumented platform surface

**Adjacent tools with no coverage at all** — [undocumented] · Four MCP tools the skill
never mentions: `tenant_ai_sms_agents`, `tenant_ai_followup`,
`tenant_ai_first_day_followup`, `tenant_ai_webhooks`.

**Webhook definition** — [undocumented] · Example 01 invokes `webhook_dispatcher` with
`webhook_id = 64`, but nothing explains how webhooks are defined or how to find an id.
`tenant_ai_webhooks` is the tool. Also undocumented: the `webhook_dispatcher` override
params (`webhook_id`, `webhook_id_description`, `webhook_sentence`,
`webhook_sentence_description`).

**`prefix` on the Voice Prompt** — [unknown] · A `text` field described only as "Prompt
prefix." Null on the tenant inspected. If it prepends to the prompt it interacts directly
with the caching guidance in `architecture.md`.

**`message` on an Intro** — [unknown] · "Optional AI text-response message." Likely the
SMS/text counterpart to the voice script; unverified.

**`inbound_number` and multi-number tenants** — [undocumented] · Intros bind to a
specific inbound number. Nothing covers running several inbound numbers, or how number +
tag interact when selecting an Intro.

**Conversion accounting** — [undocumented] · `call_transfer_buffer` (60s on the tenant
inspected) is "seconds after a transfer required to count a call as a conversion." That is
a business metric that should shape how transfer sentences are written — a transfer that
drops at 30s doesn't count. No guidance mentions it.

**DNC and opt-out mechanics** — [undocumented] · `opt_out_tags` moves contacts to DNC
based on call-result tags (`DO_NOT_QUALIFY`, `NOT_INTERESTED`, `USER_ENDED_CALL`), and
`hangup_block_limit` / `sip_block_threshold` do the same on repeated failures. Empty and
null respectively on the tenant inspected. Script wording determines which tag a call
earns, so this is prompt-adjacent, not pure ops.

**Meeting/calendar settings** — [undocumented] · `meeting_same_day`,
`ai_mode_of_meeting` (`call` | `in-person`), `ai_calender_uuid`, and `timezone` all affect
scheduling scripts. Both `ai_calender_uuid` and `timezone` were **null** on a tenant that
books meetings — worth understanding before writing another scheduling script, since a
null timezone and an ISO-8601 question are a bad combination.
`ai_calender_uuid` is writable through `tenant_ai_settings` as a free string (no enum),
but the MCP has no calendar tool: it can't create, list or read calendars. The calendar
has to be created in the Bigly app and its UUID handed over, and nothing checks that a
value points at a real calendar — a typo is accepted silently.

**OOO appointments on a `transfer_call` tenant** — [unverified] · The OOO Intro offers an
appointment with no booking tool; post-call processing is believed to tag the contact
`MEETING_BOOKED` and put it on the AI calendar. The skill leaves `prompt_type` at
`transfer_call` and adds no tool to the Voice Prompt. Lautaro is testing this on a test
tenant (2026-10-26). If appointments don't land, revisit `prompt_type`
(`meeting_scheduler`?) and the tool list in `ooo-and-callback.md` and `SKILL.md`.

**Audit reminder cadence** — [decision] · The skill can't count runs, so the "audit
this tenant" reminder fires once per conversation, after the first write. A real "every
N runs" would need the MCP to count writes per tenant and return a hint for the skill
to relay.

**Call caps and dialing filters** — [undocumented] · `max_ai_call_per_day` / `_week` /
`_month`, `verify_number_before_call` (blocks landlines), `allowed_zip_codes`,
`default_zip_code`.

**Infrastructure selection** — [undocumented] · `live_kit_version` schema says
"recommended: `live_kit_gateway`" while the tenant inspected runs `self_hosted`. Also
`sip_provider`, `aegis_workflow_id`, `ping_post_service`. Probably out of prompting scope,
but the recommendation mismatch should be a known thing rather than a surprise.

**Model choice** — [undocumented] · The catalog offers GPT-5 Mini/Nano and Claude
Sonnet 4.6 alongside the GPT-4.1 Mini default. `defaults.md` now says switching is
deliberate, but nothing says *when* it would be justified, or which of this skill's
conventions (literal instructions, ALL CAPS branches, explicit expected answers) are
GPT-4.1-Mini-specific scaffolding that a stronger model wouldn't need.

**`voice_emotion` as a real lever** — [undocumented] · 58 values, applied to the whole
call. The skill correctly says tone is TTS-owned and unreachable from the prompt, but
never says it's configurable per prompt.

**Multilingual config** — [undocumented] · `speech_language: multi`,
`turn_detection_language: multilingual`, `voice_language: es`. Example 01 branches Spanish
transfers in prompt text, but nothing connects that to the config fields that make
non-English actually work.

---

## Script-writing gaps

**Compliance beyond the recording disclosure** — [decision] · The skill enforces "on a
recorded line" and nothing else: no two-party-consent guidance, no opt-out phrasing, no
DNC language. Either that's deliberately out of scope or it's the largest legal gap in a
skill that writes outbound cold-call scripts. Worth an explicit call either way.

**"Are you AI?"** — [undocumented] · Example 01 answers `"Yes"` and doesn't volunteer it.
That's a compliance-adjacent default with no guidance behind it.

**Hostility and profanity handling** — [undocumented] · Example 01 uses a literal
blocklist plus a three-strikes rule for immediate silent `end_call`. Partly overlapping
with the tenant's `end_call` tool description. No standard block exists in
`global-rules.md`.

**Mid-flow computation** — [undocumented] · Example 01 does arithmetic on collected
answers to gate qualification. Nothing covers where calculations belong or how to keep
them out of TTS output.

**Rebuttals** — [undocumented] · One paragraph in `SKILL.md`. The two-attempt push on
the opening NO is now a stated convention, but there is still no example of the
rebuttal content itself — what the bot should actually say on each attempt.

**Injectable contact attributes** — [undocumented] · The `# General Rule` block now
specifies the variable section, but no list of injectable contact attributes beyond
`{name}` exists.

**Multi-campaign tenants** — [undocumented] · Company variables are tenant-wide, so two
campaigns needing different bot names or goals cannot both use `{bot_name}`.
`variables.md` now flags the hazard; there's no documented pattern for resolving it.

**Testing before going live** — [undocumented] · `tenant_ai_call_context_preview` is now
documented as the verification step, but there's no procedure beyond it: no test-call
practice, no way to exercise branches, no acceptance criteria.

---

## Examples

**One example, one shape** — [decision] · `examples/01` is outbound qualification →
transfer. Nothing exists for a scheduling script, an inbound-only flow, a full OOO Intro,
or a bilingual script end-to-end.

**Example 01 is missing a layer** — [undocumented] · `examples/README.md` requires "every
layer that exists for it," but 01 has no Voice Prompt *configuration* values — only prompt
text. A reader can't see what settings that production script actually ran with.

---

## Open questions

Small, and each answerable with one MCP read or one question to someone who knows the
platform:

- Is `{first_name}` a valid attribute, or is `{name}` the only spelling?
- What is `prefix` for?
- Does `uses_prompt_attributes` gate only `{}` substitution, or `[]` handling too?
- What is the full list of injectable contact attributes?
- What does `min_silence_duration` actually do, and what should its default be?
- Does an Intro's `questions_inbound` fall back to `questions` when empty, or does an
  inbound call with no inbound questions simply ask nothing?
