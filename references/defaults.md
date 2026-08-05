# Default Voice Prompt Configuration

Unless a specific use case requires different values, use these defaults for every
Voice AI prompt.

Every setting on this page lives on **`tenant_ai_voice_prompt`** over the platform MCP
(*not* `tenant_ai_settings`, which is tenant plumbing). The `MCP field` column is the
key to send. Read the tool first — its response includes the catalog of valid
`ai_*_id` values. See `platform-mcp.md`.

## Call Settings

| Setting               | MCP field                  | Default Value                                                   |
| --------------------- | -------------------------- | --------------------------------------------------------------- |
| Maximum Call Duration | `max_duration_of_call`     | Depends on the script length, but **never less than 4 minutes** |
| Maximum Ringing Time  | `max_ringing_time`         | 40 seconds                                                      |
| AI Speak Duration     | `idle_user_speak_duration` | 7 seconds                                                       |
| Silence Timeout       | `silence_timeout`          | 30 seconds                                                      |
| Allow Interruptions   | `allow_interruption`       | `true`                                                          |

`max_duration_of_call` is in **seconds** (range 30–1800), so the 4-minute floor is
`240`. `max_ringing_time` caps at 99.

## Background Audio

| Setting                 | MCP field                 | Default Value     |
| ----------------------- | ------------------------- | ----------------- |
| Background Sound        | `background_audio_file`   | `CallCenterNoise` |
| Background Sound Volume | `background_audio_volume` | 0.03              |

## Speech-to-Text (STT)

| Setting     | MCP field                    | Default Value                      |
| ----------- | ---------------------------- | ---------------------------------- |
| Provider    | —                            | Bigly Sales                        |
| Model       | `ai_speech_to_text_model_id` | Bigly Sales Parakeet STT (id `12`) |
| Language    | `speech_language`            | `en-US`                            |
| Use Context | `use_context`                | `true`                             |
| Key Terms   | `key_terms`                  | empty — but see below              |

`key_terms` is a comma-separated list of terms to bias transcription toward. It has no
default, and it is the correct fix when the bot mishears a product, company, or drug
name — not a prompt rule. Fill it whenever the script contains proper nouns the caller
will say back.

## Text-to-Speech (TTS)

| Setting  | MCP field        | Default Value                       |
| -------- | ---------------- | ----------------------------------- |
| Provider | —                | Cartesia                            |
| Voice    | `ai_voice_id`    | Sierra - California Girl (id `106`) |
| Language | `voice_language` | `en`                                |
| Speed    | `voice_speed`    | 1                                   |
| Volume   | `voice_volume`   | 1                                   |
| Emotion  | `voice_emotion`  | `neutral`                           |

`voice_volume` has a floor of 0.01 — it cannot be set to 0. `voice_emotion` accepts 58
values; `neutral` is the default for a reason, since a fixed emotion applies to the
whole call regardless of what is being said.

## Large Language Model (LLM)

| Setting     | MCP field                    | Default Value          |
| ----------- | ---------------------------- | ---------------------- |
| Type        | —                            | Azure                  |
| Model       | `ai_large_language_model_id` | GPT 4.1 Mini (id `11`) |
| Temperature | `temperature`                | 0.7                    |
| Top P       | `top_p`                      | 0.9                    |
| Max Tokens  | `max_tokens`                 | 150                    |

The catalog also carries GPT-5 Mini/Nano and Claude Sonnet 4.6. Azure GPT 4.1 Mini
stays the default because the conventions in this skill — literal instructions, ALL
CAPS branch keywords, explicit expected answers — are tuned for it. Changing model is a
deliberate decision, not a default.

## Turn Detection

| Setting                    | MCP field                   | Default Value  |
| -------------------------- | --------------------------- | -------------- |
| Provider                   | `turn_detection_provider`   | `livekit`      |
| Language                   | `turn_detection_language`   | `english`      |
| Interrupt Minimum Words    | `interrupt_min_words`       | 4              |
| Interrupt Speech Duration  | `interrupt_speech_duration` | 0.5 seconds    |
| Minimum End Pointing Delay | `min_end_pointing_delay`    | 0.80 seconds   |
| Maximum End Pointing Delay | `max_end_pointing_delay`    | 3 seconds      |
| Minimum Silence Duration   | `min_silence_duration`      | *undocumented* |

`min_silence_duration` exists on the platform but has never had a documented default
here; a live tenant runs `0.5`. Treat it as matching whatever is already set rather
than changing it, and flag it if a turn-detection problem is being investigated.

## Conversion path

| Setting         | MCP field     | Values                                 |
| --------------- | ------------- | -------------------------------------- |
| Conversion path | `prompt_type` | `transfer_call` \| `meeting_scheduler` |

No default — it must match the script. A transfer script with `prompt_type` set to
`meeting_scheduler` is a real misconfiguration, not a cosmetic one.

---

# Default Opening Sentences (Voice Prompt Configuration)

The Voice Prompt Configuration includes default opening sentences that are separate
from the prompt's Intro section. Configure them as follows.

## Outbound Calls

| Setting                             | Default Value                         |
| ----------------------------------- | ------------------------------------- |
| First Outbound Sentence             | `"Hi, can I please speak to {name}?"` |
| Second Outbound Sentence (Optional) | `""` (leave empty)                    |

## Inbound Calls

| Setting                | Default Value                    |
| ---------------------- | -------------------------------- |
| First Inbound Sentence | `"Hi, thanks for calling back."` |

These serve as **fallbacks**. At execution time they are overridden by the opening
sentences defined in the prompt's Inbound Intro or Outbound Intro sections. Leave
them unchanged unless there is a specific technical requirement to modify them.

Note these defaults carry no recording disclosure, and that is correct — the
outbound fallback is a bare identity check, and the **"on a recorded line."**
requirement is satisfied by the Intro on the bot's first substantive turn. Don't
"fix" these sentences by adding the phrase here; verify it in the Intro instead.
