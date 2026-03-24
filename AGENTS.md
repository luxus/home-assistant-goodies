# Agents — Home Assistant Blueprints Repository

## Overview

This repository contains Home Assistant automation blueprints. All files are YAML — no
Python, no tests, no build system. The primary artifact is `blueprints/automation/*.yaml`.

---

## Build / Lint / Test Commands

There is no automated tooling in this repository. Before committing:

- **YAML syntax** — visually scan for consistent indentation (2-space), correct `!input`
  references, and valid Jinja2 template expressions
- **Import validation** — paste the raw GitHub URL into Home Assistant:
  `Settings → Automations → Blueprints → Import from URL`
- **Manual test** — trigger the automation manually (e.g. toggle the trigger entity)
  and observe the full action sequence in `Developer Tools → Trace`

No CI/CD pipeline exists yet.

---

## Code Style

### YAML Structure

- 2-space indentation, always
- `!input` references at the top of `trigger:`, `variables:`, `data:`, and `action:`
  blocks — never inside Jinja2 template strings
- `mode:`, `description:`, `alias:` fields on the automation root — not inside
  `blueprint:`
- Blank lines between top-level blocks (`blueprint:`, `trigger:`, `variables:`,
  `conditions:`, `actions:`, `mode:`)

### Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Blueprint file | `snake_case.yaml` | `power_monitor_espresso_ready.yaml` |
| Blueprint `name` | Title case, hyphenated | `Power Monitor - Sustained Threshold Tracker` |
| Automation `alias` | Title case, hyphenated | `Espresso Ready Roast` |
| Entity IDs | `snake_case` | `sensor.smart_plug_v2_89b92a_power` |
| `scene_id` | `snake_case` | `speaker_before_tts_snapshot` |
| `task_name` | Sentence case, quoted | `"blueprint generated announcement"` |
| Input names | `snake_case` | `target_boolean`, `duration_minutes` |

### Trigger Variables

Always expose `!input` values as Jinja-accessible trigger variables using `variables:` on
the trigger block — this avoids `!input` being used inside template strings, which is
invalid in HA blueprints:

```yaml
trigger:
  - trigger: state
    entity_id: !input power_sensor
    variables:
      threshold: !input threshold          # now accessible as {{ threshold }}
      target_boolean: !input target_boolean  # accessible as {{ target_boolean }}
```

### Template Strings

- Use `| float(default)` for numeric state conversion — never assume a sensor returns a
  number
- Use `states('entity_id')` (function form) in conditions; `trigger.to_state.state`
  (attribute form) in trigger templates
- Quote literal strings in templates: `{{ 'playing' }}` not `{{ playing }}`
- Avoid complex arithmetic inside templates; compute in a `variables:` block instead

### Action Ordering (TTS Announcement Pattern)

When implementing a media player TTS announcement with state restoration, follow this
exact sequence:

1. `ai_task.generate_data` — generate the text
2. `variables:` — capture `was_playing`, `old_volume` **after** generation
3. `scene.create` — snapshot full player state
4. `media_player.media_pause` (if `was_playing`) + delay
5. `media_player.volume_set` — duck to announcement level
6. `media_player.play_media` — with `media:` block, `media_content_type: music`
7. `wait_template` for playing to start
8. `wait_template` for playing to stop
9. `media_player.volume_set` — restore volume explicitly (fast, before scene restore)
10. `scene.turn_on` — restore full scene state
11. `media_player.media_play` (if was playing) + delay

### Comments

- Block comments (`#`) for top-level sections and action block purposes
- Inline comments for non-obvious Jinja logic or intentional delays
- Keep comments on their own line — not at end of lines with code
- Document every `choose:` branch with its triggering condition

### Error Handling

- No try/catch in YAML; rely on HA's built-in timeout and condition re-checks
- Always add a `condition: template` re-check before any irreversible action (e.g. after
  a `delay:` in a state-dependent sequence)
- Use `timeout:` on all `wait_template:` calls as a safety valve
- `mode: single` prevents concurrent executions; use it for stateful sequences

### Input Selectors

Use the narrowest selector possible:

```yaml
selector:
  entity:
    domain: media_player   # not just entity: {}
  number:
    min: 0.0
    max: 1.0
    step: 0.05
    mode: slider           # always specify mode for number inputs
  text:
    multiline: true        # always specify for multi-line prompts
```

Always provide a `default:` value for optional inputs. For `ai_prompt`, provide a
sensible placeholder that makes the blueprint usable immediately after import.

### Blueprints vs Plain Automations

Use a `blueprint:` block only when the automation needs to be parameterized and
reusable. One-off automations (hardcoded entity IDs) should be plain YAML files in
an `automations/` directory — no `blueprint:` header, no `!input` references.

---

## File Layout

```
{repo_root}/
├── README.md
├── AGENTS.md                    ← you are here
└── blueprints/
    └── automation/
        ├── ai_contextual_tts_announcer.yaml
        └── power_monitor_espresso_ready.yaml
```

---

## Commit Conventions

Format: `type: description`

| Type | When to use |
|---|---|
| `feat:` | New blueprint |
| `fix:` | Bug fix in an existing blueprint |
| `docs:` | README, comments, documentation |
| `refactor:` | Structural change without behavior change |

Example: `feat: add power monitor sustained threshold tracker blueprint`
