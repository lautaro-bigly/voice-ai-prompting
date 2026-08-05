# Prompt Variables

`{}` is a real runtime injection — the platform substitutes a value before the LLM
sees the text. `[]` is not an injection; it tells the model to look up something it
already has in context (`"Hello [user first name]"`). Getting these backwards is a
common bug: `[first_name]` is never substituted, and the bot may read the brackets
aloud or improvise a name.

> **Read the tenant's real list.** Substitution only happens at all when the tenant
> setting `uses_prompt_attributes` is `true`. The authoritative list is
> `tenant_ai_company_variables` action `read` over the platform MCP — check it rather
> than trusting this table, because the set is per-tenant.

## Company variables (author-defined, per tenant)

Held in `tenant_ai_company_variables`. These are the ones you are responsible for
filling in. The list below is the set a live tenant actually exposes:

| Variable | Holds |
| --- | --- |
| `{bot_name}` | The agent's first name, as spoken. |
| `{company_name}` | The business the bot represents. |
| `{what_is_your_goal}` | One-line statement of what the call is for. |
| `{who_you_are}` | The persona — role, and relationship to the company. |
| `{who_you_are_calling}` | Who is on the other end and why they're on the list. |
| `{pricing_rule}` | The pricing restriction. |
| `{additional_instructions}` | Free-text catch-all appended to the prompt. |
| `{tenant_external_api_endpoint}` | External API endpoint, for tenants that call out. |

These are **tenant-wide, not per-prompt** — changing `{bot_name}` changes it on every
script the account runs. That is a real hazard when a tenant runs several campaigns:
if two campaigns need different bot names, they cannot share a company variable, and
the name has to be written literally into each Intro instead.

Several of these commonly sit empty on a live tenant (`who_you_are`,
`what_is_your_goal`, and `additional_instructions` were all blank on the account this
was compiled from). An empty variable resolves to nothing and silently removes
whatever the prompt said around it — read them before writing a script that leans on
one, and fill the blanks as part of the build.

Note `{working_hours}` is **not** a company variable, despite being used by the
Callback Rule and the OOO body. Confirm where the tenant supplies it before relying
on it — a `{working_hours}` that resolves to nothing turns the callback rule into a
request to pick a time from an empty list.

## Runtime values

Supplied by the platform, not by you.

| Variable | Holds |
| --- | --- |
| `{date}` | Today's date at call time. Anything reasoning about "later today", scheduling, or "in the past" needs it in scope. |
| `{intro}` | The selected Intro's `content_outbound` or `content_inbound`, substituted into the Voice Prompt. This is the join between the two layers — a Voice Prompt missing `{intro}` runs with no conversation script at all. |

`{date}` is the one runtime value worth injecting deliberately: the Callback Rule and
any ISO-8601 timestamp question are both wrong without it.

## Contact attributes

Per-contact fields from the record being called.

| Variable | Holds |
| --- | --- |
| `{name}` | The contact's name. This is the spelling live production opening sentences use (`"Hi, may i please speak to {name}?"`). Prefer it. |
| `{first_name}` | Appears in the written conventions but not in any live sentence inspected. Verify before using. |

Contact attributes are also where question answers land — a configured question writes
to `contact.<field>` (see `platform-mcp.md`), so those fields become available to later
prompts on the same contact.

Inject each one **once**, in a single `# User Personal Information` section near the
end of the prompt, and refer to it with `[]` everywhere else — this keeps the static
prefix cacheable. See `architecture.md`.

## Questions

`{Q1}`, `{Q2}`, … are the configured questions, injected where the flow asks them.
Unlike contact attributes these are used once each, so inline injection is correct
and there is no reason to collect them into a section.

Only configured questions store structured answers, and each one carries its own
`instruction` and `attribute_name` as structured fields rather than prompt prose — see
`platform-mcp.md`. An inline `Ask: "..."` is fine for a conversational aside but
captures nothing downstream.
