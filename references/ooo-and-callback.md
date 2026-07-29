# Out of Office and Scheduled Callback Prompts

## Out of Office (OOO)

Configure an OOO prompt for **almost every script**. The exception is **scheduling**
scripts: they book appointments rather than transferring to a live call center, so
business hours are irrelevant and no OOO prompt is needed.

### Configuration

OOO is for **inbound calls outside business hours only**.

- Leave the **Outbound** section empty.
- Configure only the **Inbound** section.

The inbound opening sentence must clearly say the office is closed:

```text
Thanks for calling {company_name}. This is {bot_name}. Our office is currently closed, but you can reach us during our regular business hours, and we'll be happy to assist you.
```

Wording can vary, but it must always tell the caller the office is closed and
invite them to call back during business hours.

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

### Placement of the callback pitch

The **Second Outbound Sentence** in the Voice Prompt Configuration is optional and
usually left empty. Because of that, the callback-specific introduction is normally
placed directly in the **Intro** prompt rather than in the configuration.

When cloning, review **both** the opening sentence and the body of the Intro — any
wording that references why you're calling needs to acknowledge the previously
requested callback, not just the first line.
