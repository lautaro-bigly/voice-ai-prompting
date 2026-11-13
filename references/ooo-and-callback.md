# Out of Office and Scheduled Callback Prompts

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

Configure an OOO prompt for **every transfer script**, and for no other kind. Working
hours and the OOO prompt both exist for one reason: live agents answer the phone during
some hours and not others. A script that does not transfer — scheduling, information,
anything else — has no live agents, so business hours are irrelevant, no OOO prompt is
needed, and working hours must never be set for it.

### What it does

A caller outside business hours gets the **same qualifying flow** as during business
hours. Only the ending changes: there is no live agent to transfer to, so a caller who
qualifies is offered an **appointment**, and an agent reaches them during working hours.

The OOO Intro never tells the caller the office is closed, never asks them to call back,
and never offers a callback. An appointment is not a callback — the Callback Rule in the
Voice Prompt is for a caller who is busy, and still applies here on its own terms.

There is no booking tool — not here, not on any script, not on either engine. The bot
offers the appointment and agrees a day and time; after the call a separate layer reads
the transcript, tags the call `MEETING_BOOKED` and puts the appointment on the tenant's
AI calendar. Never add a tool for it. Whether it also works with `prompt_type` left at
`transfer_call` is the one open point — see `GAPS.md`.

### Calendar prerequisite — the user's job

The appointment lands on the calendar selected for AI meetings. If there is none, it has
nowhere to go. **The skill cannot set this up**: the MCP can't create or list calendars.
Tell the user plainly that they have to create and configure it themselves, in the Bigly
app, before the OOO Intro goes live:

1. **Calendars → Availability → Add availability**, and configure the calendar.
2. **AI tab → Configure → Calender For AI Meetings**, and pick that calendar.

Make sure they understand this is their step, not one the skill took. Once they say it's
done, read `ai_calender_uuid` with `tenant_ai_settings` — it is `null` until a calendar
is picked.

### Configuration

OOO is for **inbound calls outside business hours only**.

- Leave the **Outbound** section empty (`content_outbound`, `first_outbound_sentence`).
- Configure the **Inbound** section (`content_inbound`, `first_inbound_sentence`,
  `questions_inbound`). The questions are the main Intro's `questions_inbound`, with the
  same `attribute_type`, `attribute_name` and expected output.
- Set `tag` to `after_office_hours`.
- **Never set `inbound_number`.** It stays empty under every circumstance. A number
  bound to the OOO Intro makes every call to that number get the OOO prompt, inside
  working hours too — callers are offered an appointment while agents are waiting.
  The `after_office_hours` tag alone is what selects it, outside business hours.
- `prompt_type` stays `transfer_call`. The OOO appointment does not make the tenant a
  scheduling script — pending the test noted in `GAPS.md`.

If an existing OOO Intro has an `inbound_number`, clear it whenever you touch the
tenant, whatever the task — without asking whether to. It is a workflow-breaking
misconfiguration, not a choice. The removal still shows in the `preview_update` diff
like any other field.

The inbound opening sentence is the standard inbound one. It carries the recording
disclosure and says nothing about the office being closed:

```text
Thank you for calling {company_name}. This is {bot_name} on a recorded line. How can I help you today?
```

### Default OOO body

Clone the main Intro's inbound body — Objective, Initial Ask Handling, Qualification
Flow, FAQs, Rebuttal — and change only this:

- The Objective says the goal is an appointment with a specialist, not a transfer.
- `# Transfer` is replaced by `# Appointment` below, and every `proceed to #Transfer`
  points at `#Appointment`.

```text
# Appointment

Transfers are not available on this call. Never invoke the transfer_call tool.

Say: "Great, I'd like to set up an appointment so one of our specialists can reach out to you."

Ask: "What day works best for you?"
(You must receive a day within [working hours], not in the past.)

Ask: "What time works best for you that day?"
(You must receive a time within [working hours]. Never ask for the time zone; take the time exactly as the user says it.)

Repeat the day and time back and ask: "Does that work for you?"

If YES or EQUIVALENT INTENT, invoke the end_call tool. Parameters:
-end_sentence:"Perfect, you're all set. One of our specialists will reach out to you then. Have a great rest of your day!"
If NO or EQUIVALENT INTENT, ask again for the day.
If the user doesn't want an appointment, invoke the end_call tool. Parameters:
-end_sentence:"No problem at all. Thanks for calling, and have a great rest of your day!"
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
