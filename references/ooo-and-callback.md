# Out of Office and Scheduled Callback Prompts (only used for live transfer campaigns)

Neither of these is a separate feature. **Both are ordinary Intros distinguished by
their `tag` field**, which the platform matches against the contact's latest tag:

| Prompt | Intro `tag` |
| --- | --- |
| Out of Office | `after_office_hours` |
| Scheduled callback | `SCHEDULED_CALLBACK` |
| Main script | `null` |

So "create an OOO prompt" means creating an Intro tagged `after_office_hours` via
`tenant_ai_intros`. Everything that applies to an Intro — the two content fields, the
opening sentences, the questions array — applies to these too. See `platform-mcp.md`
for the full tag list, which includes cases this file doesn't cover
(`TRANSFERRED_DROPPED`, `MEETING_BOOKED`, `SMS_CALLBACK`).

## Out of Office (OOO)

Configure an OOO prompt for **almost every script**. The exception is **scheduling**
scripts: they book appointments rather than transferring to a live call center, so
business hours are irrelevant and no OOO prompt is needed.

### Configuration 

OOO is for **inbound calls outside business hours only**. (why don't we qualify and only then "callcenter is closed")

- Leave the **Outbound** section empty (`content_outbound`, `first_outbound_sentence`).
- Configure only the **Inbound** section (`content_inbound`, `first_inbound_sentence`).
- Set `tag` to `after_office_hours`.

The inbound opening sentence must clearly say the office is closed, and — like every
other flow — must carry the recording disclosure:

```text
Thanks for calling {company_name}. This is {bot_name} on a recorded line. Our office is currently closed, but you can reach us during our regular business hours, and we'll be happy to assist you.
```

Wording can vary, but it must always tell the caller the office is closed, invite
them to call back during business hours, and include **"on a recorded line."** An
after-hours call is recorded like any other, so the disclosure is not optional here.

### Default OOO body

Unless the use case requires different behavior:

```text
Objective:
To advise the user that the call center is currently closed and that they should call back during business hours. After providing this information, invoke the "end_call" tool. Never say "end_call" or explicitly mention that the call is being ended.

Who you are:
{who_you_are}

Who you are calling:
{who_you_are_calling}

Company name:
{company_name}

Current date:
{date}

[RULES]

Say:
"The following are our working hours: {working_hours}. Please let me know when you would like us to call back."
```

---

## Scheduled Callback Prompt

Some contacts are unavailable when the AI first calls and request a callback.
**Always create a Scheduled Callback prompt.**

Build it by **cloning the standard outbound Intro** and adjusting the wording to
reflect that this is a callback, not initial outreach. The conversation flow,
qualification logic, transfer logic, and scheduling behavior stay the same — only
the wording changes. The point of a callback Intro is to qualify a user who
couldn't get through the original Intro, so the qualification must survive intact.

Example opening:

```text
Hi, this is {bot_name} with {company_name} on a recorded line. You requested that we call you back...
```

Set `tag` to `SCHEDULED_CALLBACK`.

### Placement of the callback pitch

Two different fields share almost the same name, and the distinction decides where this
goes:

- `outbound_second_message` — Voice Prompt level, a global fallback. Normally left null.
- `second_outbound_sentence` — **Intro** level. This is where the pitch goes.

Because the Voice Prompt fallback stays empty, the callback-specific introduction is
placed on the Intro. In production the outbound pitch — including the recording
disclosure — is carried by the Intro's `second_outbound_sentence`, with
`first_outbound_sentence` doing nothing but asking for the contact by name.

When cloning, review **both** the opening sentence and the body of the Intro — any
wording that references why you're calling needs to acknowledge the previously
requested callback, not just the first line.
