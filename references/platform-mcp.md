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
inbound number.

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

- `attribute_type` — `boolean`, `string`, or `number`.
- `attribute_name` — the contact field the answer lands in (`contact.<field>`). This is
  what makes an answer available downstream; an inline `Ask: "..."` in the body stores
  nothing.
- `instruction` — a real field, and it carries two different jobs. For a plain question
  it states the expected answer (`"must get a yes, no or equivalent answer"`). For a
  question that must normalize free speech it holds the mapping rule:

```text
Interpret the user's answer and map it to exactly one of these values:
"Single Family"
"Multi Family"
"Apartment"
```

```text
Convert the user's input into a normalized ISO 8601 UTC timestamp (RFC 3339),
formatted exactly as YYYY-MM-DDTHH:MM:SSZ, using UTC time with the Z suffix.
```

Because `instruction` exists as a field, the parenthetical convention in `SKILL.md`
(`(You must receive a YES/NO answer.)`) is the *body-side* echo of it — the
authoritative expected-answer rule belongs in `instruction`, and the body reference
keeps the flow readable. Set both.

## Conversion path

`prompt_type` on the Voice Prompt is an enum: `transfer_call` or `meeting_scheduler`.
This is the platform-level counterpart to the transfer-vs-scheduling split in
`SKILL.md` — set it to match the script, and on `meeting_scheduler` keep `transfer_call`
out of the tool list entirely.

## Configurable tool overrides

`tenant_ai_tools_configuration` covers exactly two tools:

- `end_call` — `description` (when to fire) and `default_sentence`.
- `webhook_dispatcher` — `webhook_id`, `webhook_sentence`, and a description for each.

`transfer_call` is **not** configurable here; its `transfer_number` and
`transfer_sentence` live in the Intro body. Tenants may also reference other tools from
prompt text (`ping_post_meeting`, `ping_post_meeting_book` appear in production
scheduling prompts), so read the existing prompt before assuming the tool list.

## Verify before you finish

`tenant_ai_call_context_preview` assembles the final prompt, first/second message,
qualifying questions and pipeline config exactly as a real call would, without dialing.
It takes `direction`, `intro_id`, `contact_id`, `test_name`, `service_provider` — all
optional. Run it after any change and read the assembled output: it is the only way to
confirm `{intro}` and every `{variable}` actually resolved instead of reaching the LLM
as literal braces.
