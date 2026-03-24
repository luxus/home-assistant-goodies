# Home Assistant Goodies

A collection of Home Assistant blueprints and automations.

---

## Blueprints

### AI Contextual TTS Announcer (HomePod Safe)

Generates an AI-written announcement and plays it on a media player with full volume ducking and queue restoration. Designed to be safe for HomePods and other picky speakers.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fluxus%2Fhome-assistant-goodies%2Fmain%2Fblueprints%2Fautomation%2Fai_contextual_tts_announcer.yaml)

#### What it does

1. Watches a trigger entity for an `on` state
2. Calls an AI Task entity to generate a custom spoken message from your prompt
3. Snapshots the current speaker state (volume, playback, queue)
4. Pauses playback if something was playing
5. Ducks the volume to your announcement level
6. Plays the AI text through your TTS engine
7. Waits for the announcement to finish
8. Restores the original volume and scene
9. Resumes playback if it was interrupted

#### Requirements

| Requirement | Notes |
|---|---|
| Home Assistant 2024.6+ | Required for `ai_task` domain support |
| AI Task integration | OpenAI, Google Generative AI, Nvidia NIM, etc. |
| TTS integration | ElevenLabs, Piper, Google Cloud TTS, etc. |
| Media player | HomePod (via AirPlay), Sonos, Chromecast, etc. |

#### Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| Trigger Entity | Any entity that transitions to `on` | Yes | — |
| Target Speaker | The `media_player` to announce on | Yes | — |
| Announcement Volume | Volume level during TTS (0.0–1.0) | No | `0.75` |
| AI Task Entity | The `ai_task` entity for text generation | Yes | — |
| AI Prompt | Instructions for the AI — what to say and how | Yes | Friendly placeholder |
| TTS Entity | The `tts` entity to render speech | Yes | — |

#### Example: Espresso Barista Roast

Trigger: a smart plug powering an espresso machine has been on for 10 minutes (the boiler is fully heated).

```yaml
alias: Espresso Ready Roast
triggers:
  - trigger: state
    entity_id: switch.espresso_machine
    to: "on"
    for:
      minutes: 10
```

AI Prompt:
```
Write a short, sarcastic one-liner mocking a pretentious but untalented
third-wave home barista. The joke must center around the fact that the
espresso boiler is fully heated and waiting on them.
Output ONLY the raw spoken text. Do not include any formatting,
quotation marks, or conversational filler.
```

#### Notes

**HomePod quirks**
HomePods via AirPlay can take a moment to re-establish a session after an interruption. The 1s pre-announcement delay and 2s post-restoration delay are intentional — remove them if your speaker is faster.

**Volume restore**
Volume is restored in two steps: an explicit `volume_set` call for immediate effect, followed by `scene.turn_on` which restores the full player state. This is intentionally redundant to handle edge cases where scene restore takes time.

**`mode: single`**
Only one instance of the automation runs at a time. If the trigger fires again mid-announcement, it is dropped. Change to `queued` in the automation editor if you want announcements to line up.

**AI prompt tips**
- End your prompt with: *"Output ONLY the raw spoken text. No quotes, no formatting."*
- Keep expected output under ~200 words to stay within TTS comfort zones.
- You can inject sensor values into the prompt using a template in the automation that calls this blueprint.
