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
  LLM. (Parakeet.) You never prompt it directly, but its failure modes are worth
  knowing, because the prompt has to absorb them — see the STT note below.
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

## The reachable edge of an unreachable component

The line falls around each component's *internals*, not around its input and output.
The LLM sits between STT and TTS, so it controls the text crossing both boundaries
even though it controls neither engine. Two consequences worth knowing, because both
look like violations of the rule above:

**TTS input — pronunciation is reachable by respelling.** You cannot ask Cartesia to
pronounce something differently, but you choose the characters it reads. A script
whose numbers come out wrong is fixed with a prompt rule:

```text
Never output digits or numeric symbols in any assistant response. Always write
numbers in spoken word form, in the language currently being used.
```

The same trick handles a mispronounced company or drug name — spell it phonetically
in the prompt and let TTS read the respelling.

**STT output — trust is reachable from the prompt; accuracy is reachable only from
config.** No prompt rule makes Parakeet transcribe better, but two separate levers
exist and they are often both needed.

The config lever is `key_terms` on `tenant_ai_voice_prompt` — a comma-separated list
that biases transcription toward specific terms. If the bot mishears your company name,
a product, or a drug name, that is the first fix, and no amount of prompting substitutes
for it.

The prompt lever is telling the LLM which parts of a transcript to disbelieve. Proper
nouns are the usual casualty, so production scripts instruct the model to decide on
intent rather than on the literal words:

```text
STT may mishear names, so do not trust names in the transcript. Decide what to do
based on intent only. Ignore name mismatches — if we called Jim and the transcript
says "this is Jen," treat it as confirmation and continue.
```

Without that rule the bot hangs up on people who answered their own phone. Set
`key_terms` *and* the trust rule — the first reduces how often the name is mangled, the
second stops a mangling from ending the call.

The general move: when an instruction targets a component you can't reach, restate it
as an instruction about text the LLM *produces* or *consumes*. If it can't be
restated that way — response delay, background noise, voice selection — it belongs in
config and nowhere else.

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
