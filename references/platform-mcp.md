# Driving the Platform Over MCP

Each tenant exposes its own MCP endpoint (`https://<tenant>.biglysales.com/api/v2/mcp`,
server name `BiglySales MCP Server`). The account's **MCP API** feature must be enabled
tenant-side; if it isn't, every call returns HTTP 403 `The MCP API is not enabled.`
even with a valid token. A rejected *token* returns `Unauthenticated.` instead — the
two failures look alike and mean different things.

## Tool surface

Ten tools, all named `tenant_ai_*` and all acting on the active tenant.

| Tool | Use it for |
| --- | --- |
| `tenant_ai_voice_prompt` | The Voice Prompt, opening messages, and **every setting in `defaults.md`** |
| `tenant_ai_intros` | Intros — including the OOO and callback prompts, and the qualifying questions |
| `tenant_ai_company_variables` | The `{placeholder}` key/value pairs |
| `tenant_ai_tools_configuration` | `end_call` and `webhook_dispatcher` overrides |
| `tenant_ai_call_context_preview` | Assemble the final prompt without placing a call — the verification step |
| `tenant_ai_settings` | Tenant plumbing: SIP, DNC thresholds, campaign triggers, concurrency |
| `tenant_ai_webhooks` | Webhook definitions |
| `tenant_ai_sms_agents` | SMS agents |
| `tenant_ai_followup` | Followup sequence |
| `tenant_ai_first_day_followup` | First-day followup rows and their linked intro |

`tenant_ai_settings` is **not** where prompt config lives — it holds tenant-wide
plumbing. Everything in `references/defaults.md` is on `tenant_ai_voice_prompt`.

## Every write is a three-step handshake

Mutating tools take `action`, and the platform deliberately makes writes awkward:

```
read  →  preview_update  →  apply_update + confirmation
```

`apply_update` is refused without an exact confirmation string per tool:

| Tool | `confirmation` |
| --- | --- |
| `tenant_ai_voice_prompt` | `APPLY_TENANT_AI_VOICE_PROMPT_UPDATE` |
| `tenant_ai_intros` | `APPLY_TENANT_AI_INTROS_UPDATE` |
| `tenant_ai_company_variables` | `APPLY_TENANT_AI_COMPANY_VARIABLES_UPDATE` |
| `tenant_ai_settings` | `APPLY_TENANT_AI_SETTINGS_UPDATE` |
| `tenant_ai_tools_configuration` | `APPLY_TENANT_AI_TOOLS_CONFIGURATION_UPDATE` |

Always run `preview_update` and show the diff before applying. This is a live tenant —
an `apply_update` changes the prompt on calls that are placed minutes later. On
`tenant_ai_intros`, `apply_update` **with** an `id` updates that intro and **without**
one creates a new intro; omitting the id by accident silently creates a duplicate
rather than erroring.

Call `read` first on `tenant_ai_voice_prompt` regardless — the response carries the
`catalog` of valid LLM / voice / STT ids, which you need before setting any of them.

## Two layers of opening sentence, confusingly similar names

Both levels exist and the Intro's win at execution time. Don't mix them up:

| Voice Prompt level | Intro level |
| --- | --- |
| `outbound_first_message` | `first_outbound_sentence` |
| `outbound_second_message` | `second_outbound_sentence` |
| `inbound_first_message` | `first_inbound_sentence` |

The Voice Prompt pair are the fallbacks described in `defaults.md`; the Intro pair are
per-conversation and override them. In production the outbound disclosure and pitch sit
in the Intro's `second_outbound_sentence` while the Voice Prompt's
`outbound_second_message` stays null — which is why the recording disclosure is a
**first substantive turn** requirement rather than a first-sentence one.

`content_outbound` and `content_inbound` are the Intro bodies, injected at `{intro}`
in the Voice Prompt. When inbound and outbound share a flow they are simply identical.

## Voice engine is per Intro

`tenant_ai_intros` carries `voice_engine` (`cascade` | `speech_to_speech`; `null` falls
back to the voice assistant's) and `speech_to_speech_settings`. `list` and `read` return
`effective_voice_engine` — check it before editing any Intro, because the two engines
are written differently. The delegation prompt of an S2S Intro is
`speech_to_speech_settings.reasoning_instructions`; send only the keys being changed.
`tenant_ai_call_context_preview` shows the engine a call would run on under
`pipeline_config.engine`. See `speech-to-speech.md`.

## Intros carry the tag that selects them

An Intro is not only the main script. The `tag` field is matched against the contact's
latest tag name, and that is the whole mechanism behind the OOO and callback prompts —
they are ordinary Intros with a tag, not a separate feature:

| `tag` | Fires when |
| --- | --- |
| `null` | Default — the main script |
| `after_office_hours` | Inbound outside business hours (**this is the OOO prompt**) |
| `SCHEDULED_CALLBACK` | Contact previously asked to be called back |
| `TRANSFERRED_DROPPED` | A transfer that dropped |
| `MEETING_BOOKED` | Contact already has a booked meeting |
| `SMS_CALLBACK` | Callback requested over SMS |

Custom tag strings are accepted too. `inbound_number` binds an Intro to a specific
inbound number. Never set it on the `after_office_hours` Intro: the binding wins over
working hours, so every call to that number gets the OOO prompt (`ooo-and-callback.md`).

So "create an OOO prompt" means: create an Intro tagged `after_office_hours`. Two of
these tags — `TRANSFERRED_DROPPED` and `MEETING_BOOKED` — are cases the written guide
doesn't otherwise cover, and are worth asking about on a mature account.

## Questions are structured rows, not prompt text

`questions` (outbound) and `questions_inbound` are arrays of:

```json
{
  "question": "About how much do you pay for electricity each month?",
  "instruction": "must get a number",
  "attribute_name": "contact.monthly_electric_bill",
  "attribute_type": "number"
}
```

`attribute_type`, `attribute_name`, and `instruction` are **mandatory on every
question**, outbound and inbound. Never leave one out.

- `attribute_type` — `boolean`, `string`, `number`, or `list`.
- `attribute_name` — the contact field the answer lands in (`contact.<field>`). This is
  what makes an answer available downstream; an inline `Ask: "..."` in the body stores
  nothing.
- `instruction` — the question's **expected output**. Always exactly one of these
  predefined values, never free text:

| `instruction` | Use for |
| --- | --- |
| `must get a yes, no or equivalent answer` | Yes/no questions |
| `must get a number` | Amounts, counts |
| `must get a date in format Y-m-d` | Dates |
| `must get a zipcode` | Zip codes |
| `must get an answer for state and state name should be in abbreviation not full name.` | U.S. state |
| `must get an answer.` | Anything the others don't fit — names, addresses, a choice among options |

**Demo exception.** If the user explicitly says they are building a demo, confirm it
with them, then set every question's `instruction` to `must get an answer.` instead of
picking per question. Never assume a demo without that statement. `attribute_type` and
`attribute_name` are still required.

Older Intros carry free-text instructions in this field (enum mappings, ISO 8601
conversions). Don't copy them. When editing such an Intro, flag each one and replace it
with the closest predefined value; any normalization rule it held belongs in the body
parenthetical.

The parenthetical convention in `SKILL.md` (`(You must receive a YES/NO answer.)`) is the
*body-side* companion of `instruction`: the field carries the predefined expected output,
and the parenthetical carries it in readable form plus anything more specific. Set both.

## Conversion path

`prompt_type` on the Voice Prompt is an enum: `transfer_call` or `meeting_scheduler`.
This is the platform-level counterpart to the transfer-vs-scheduling split in
`SKILL.md` — set it to match the script, and on `meeting_scheduler` keep `transfer_call`
out of the tool list entirely.

`meeting_scheduler` is only this setting. It is not a tool, and no appointment tool
exists: the bot agrees a day and time and ends the call, and a post-call layer reads the
transcript, tags the call `MEETING_BOOKED` and sends it to the AI calendar. The same on
cascade and S2S.

## Configurable tool overrides

`tenant_ai_tools_configuration` covers exactly two tools:

- `end_call` — `description` (when to fire), `default_sentence`, and the description of
  its single runtime parameter, `end_sentence`.
- `webhook_dispatcher` — `webhook_id`, `webhook_sentence`, and a description for each.

`end_sentence` is the closing line the platform speaks before it hangs up. When the
script leaves it unspecified, the tool falls back to `default_sentence` — the
tenant-level default, `"Thank you for your time!"` on the tenant inspected. That
fallback is why a goodbye written as `say: "..."` followed by a bare `end_call` is
wrong even though it works: the caller hears the script's line and then the default
one. The line belongs in `end_sentence` (see `SKILL.md`, Writing conventions). The
shipped parameter description reads *"Sentence to be said when user wants to end the
call. Should be empty if not specified explicitly."*, and the tool `description` lists
the silent cases — wrong number, voicemail, IVR, profanity — as "invoke with empty
end_sentence": for those, pass an empty sentence and no message rather than leaving it
unspecified and letting the default play.

The tool also holds rows for `transfer_call` (`transfer_number`,
`transfer_call_sentence`, their descriptions, and `use_tool`), `send_sms` and
`credit_score`. S2S Intros rely on them — every tool call there uses the row's default
(`speech-to-speech.md`). On cascade, `transfer_number` and `transfer_sentence` are
written in the Intro body. Tenants may also reference other tools from
prompt text (`ping_post_meeting`, `ping_post_meeting_book` appear in production
scheduling prompts), so read the existing prompt before assuming the tool list.

## Verify before you finish

`tenant_ai_call_context_preview` assembles the final prompt, first/second message,
qualifying questions and pipeline config exactly as a real call would, without dialing.
It takes `direction`, `intro_id`, `contact_id`, `test_name`, `service_provider` — all
optional. Run it after any change and read the assembled output: it is the only way to
confirm `{intro}` and every `{variable}` actually resolved instead of reaching the LLM
as literal braces.
