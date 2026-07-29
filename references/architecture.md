# Platform Architecture and Prompting Boundaries

## Purpose

The system automates phone conversations, reducing the need for human call center
agents. Common use cases:

- Qualifying a lead before transferring them to a human agent.
- Qualifying a lead and scheduling an appointment.
- Workflows combining both qualification and appointment scheduling.

## Components

The platform is several independent components, each with a specific
responsibility:

- **LLM (Large Language Model)** — controls conversation flow, reasoning, and
  text responses. Ours is **GPT-4.1 Mini**, which is worth keeping in mind when
  prompting: it rewards explicit, literal instructions over implied ones.
- **Speech-to-Text (STT)** — converts caller speech to text before it reaches the
  LLM. The specific engine (Parakeet) doesn't matter from a prompting perspective.
- **Text-to-Speech (TTS)** — converts the LLM's text into audio. Owns voice
  characteristics: tone, pacing, pronunciation, speaking style. (Cartesia.)
- **Turn Detection** — decides when the user has finished speaking and when the AI
  should respond. (LiveKit.)

Understanding this separation is essential when writing prompts.

## What the prompt cannot do

The LLM controls the conversation and nothing else. Instructions aimed at another
component are dead text — they don't error, they're just ignored.

- **"Wait N seconds before responding"** — response timing belongs to turn
  detection, not the LLM.
- **"Ignore background noise"** — noise handling belongs to the STT engine.
- **"Use a soft voice" / "speak more slowly"** — voice quality, tone, pacing, and
  delivery belong to the TTS engine.

Focus prompts on what is actually the LLM's job: reasoning, conversation flow,
asking the right questions, following business logic, and generating appropriate
responses. Everything else is a config change in the relevant component.

## Prompt caching

There is a performance reason to concentrate dynamic values in one place.

Caching works on the **static prefix** of a prompt. The platform reuses the longest
stretch of text, from the start, that is identical to a previous call, which
reduces cost and latency on that portion. The cache breaks at the first character
that differs, so everything after a dynamic value must be reprocessed.

- **Static content** (persona, global rules, tools) is identical on every call and
  stays cacheable.
- **Dynamic per-contact values** (`{first_name}`, `{date}`) differ on every call
  and invalidate the cache from where they appear onward.

A variable scattered through the prompt creates a break point at every occurrence
and fragments the cacheable region. Injecting it once — ideally in a later
section — keeps the large static block at the front fully cacheable.

This maps onto the platform's structure: the stable **Voice Prompt** sits up front
and stays cacheable across all calls, while per-contact variables are injected in a
dedicated section rather than sprinkled throughout.

> **Note:** Exact caching behavior (minimum token thresholds, increment sizes)
> depends on the provider's implementation. Treat this as a general principle
> rather than an exact guarantee.
