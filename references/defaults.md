# Default Voice Prompt Configuration

Unless a specific use case requires different values, use these defaults for every
Voice AI prompt.

## Call Settings

| Setting               | Default Value                                                   |
| --------------------- | --------------------------------------------------------------- |
| Maximum Call Duration | Depends on the script length, but **never less than 4 minutes** |
| Maximum Ringing Time  | 40 seconds                                                      |
| AI Speak Duration     | 7 seconds                                                       |
| Silence Timeout       | 30 seconds                                                      |
| Allow Interruptions   | `true`                                                          |

## Background Audio

| Setting                 | Default Value     |
| ----------------------- | ----------------- |
| Background Sound        | `CallCenterNoise` |
| Background Sound Volume | 0.03              |

## Speech-to-Text (STT)

| Setting     | Default Value            |
| ----------- | ------------------------ |
| Provider    | Bigly Sales              |
| Model       | Bigly Sales Parakeet STT |
| Use Context | `true`                   |

## Text-to-Speech (TTS)

| Setting  | Default Value            |
| -------- | ------------------------ |
| Provider | Cartesia                 |
| Voice    | Sierra - California Girl |
| Language | English                  |
| Speed    | 1                        |
| Volume   | 1                        |
| Emotion  | Neutral                  |

## Large Language Model (LLM)

| Setting     | Default Value |
| ----------- | ------------- |
| Type        | Azure         |
| Model       | GPT-4.1 Mini  |
| Temperature | 0.7           |
| Top P       | 0.9           |
| Max Tokens  | 150           |

## Turn Detection

| Setting                    | Default Value |
| -------------------------- | ------------- |
| Provider                   | LiveKit       |
| Language                   | English       |
| Interrupt Minimum Words    | 4             |
| Interrupt Speech Duration  | 0.5 seconds   |
| Minimum End Pointing Delay | 0.80 seconds  |
| Maximum End Pointing Delay | 3 seconds     |

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
