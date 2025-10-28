## Copilot instructions for HA_blueprints

Quick orientation
- This repo contains Home Assistant blueprints (YAML automations). Primary example: `Frigate_Camera_Notifications/Beta.yaml` (large, production-ready blueprint). `Stable.yaml` is the stable variant and `README.md` documents features.
- Architecture: single-file blueprints that run as Home Assistant automations. They rely on Jinja2 templating, Home Assistant services (`notify.*`, `telegram_bot.*`, `device notify mobile_app`), and external integration with Frigate via MQTT.

What to know before editing
- YAML + Jinja: templates ({{ }}, {% %}) are embedded throughout — preserve quoting and indentation. Avoid adding or removing leading spaces in templated blocks. Many values are `!input` blueprint inputs so they appear in the blueprint editor.
- Triggers & loop: the blueprint listens to an MQTT topic (default `frigate/reviews`) and also to `mobile_app_notification_action` events. It uses `mode: parallel` and a repeat/wait_for_trigger loop that waits for review updates until the event type == `end`.
- Common variables: `event`, `id`, `review_id`, `camera`, `objects`, `sub_labels`, `attachment`, `video`, `base_url`, `client_id`. Many derived values are computed in `variables:` — change with care.
- Integrations used: Frigate (MQTT messages), Home Assistant mobile_app (device notify), `notify.<group>` targets, `telegram_bot`, and `input_text` helpers (used as a TTS id store). Default MQTT topic is configured by `mqtt_topic` input.

Project-specific patterns and conventions
- Blueprint inputs: use `!input` and the `input:` section at top of YAML. Keep the same naming and default semantics when adding new inputs.
- String formatting: templates commonly use `|replace`, `|title`, `|lower`, `|int`, `|length` and index access like `trigger.payload_json['after']['camera']`. Follow that style for consistency.
- Attachment selection: `attachment`, `attachment_2`, and `video` inputs are templates that build URLs like `{{base_url}}/api/frigate{{client_id}}/notifications/{{id}}/snapshot.jpg`. When changing image/video logic, maintain `base_url` redaction and telegram URL substitution (`telegram_base_url`).
- Debugging: there is a `debug` boolean input. When enabled the blueprint writes structured debug data to the Home Assistant logbook via `logbook.log`. There is also a `redacted` flag to hide `base_url` in output — keep it when adding new debug lines.

Examples you can reuse
- Notify mobile device (mobile_app): see `device_id: !input notify_device` block and `data:` with `image`, `video`, `clickAction`, `attachment.content-type` logic.
- Notify group/TV: a `notify.<group>` pattern is used (`notify.{{ notify_group_target }}`) and TV-specific fields (`fontsize`, `position`, `duration`, `image.url`) are set.
- Telegram fallback: `telegram_bot.send_photo` and `telegram_bot.send_video` branches replace `base_url` with `telegram_base_url` for internal access.

Editing & testing workflow
- Local edits: modify YAML files and open them in the Home Assistant Blueprint editor (Configuration → Blueprints → Import from file) to validate structure and template rendering. The `source_url` at top of the blueprint points to the upstream raw URL used for imports.
- Linting / safety checks: before pushing, validate YAML formatting (e.g., `yamllint`) and verify Jinja expressions don't break quoting. If you run Home Assistant core locally, use the built-in config check (`ha core check` or `hass --script check_config`) to validate automations.
- Runtime testing: test with a Frigate instance publishing to `mqtt_topic` (default `frigate/reviews`) or simulate MQTT payloads in Developer Tools → Services (`mqtt.publish`) using the same JSON shape the blueprint expects (look at `trigger.payload_json` usages in the YAML for schema examples).

Editing tips for AI agents
- Be conservative with structural edits. Changing indentation or removing surrounding quotes in templated values frequently breaks Home Assistant parsing.
- When altering templates, include one short example in the commit message showing input → expected rendered string (for reviewers and for automated checks).
- Preserve and update the `debug` and `redacted` logging blocks if you add new runtime information.
- When renaming an `!input` key, update every `!input` reference and the `input:` section at the top of the file.

Integration & external dependencies
- Frigate: expects review messages on MQTT. The blueprint reads `trigger.payload_json['after']` fields. Follow the MQTT shape used in `Beta.yaml` when creating test payloads.
- Mobile app notifications: uses `mobile_app` device targets and `mobile_app_notification_action` events for actionable buttons (silence/custom). The `silence-{{ this.entity_id }}` and `custom-{{ this.entity_id }}` action names are conventions used by the blueprint.
- Telegram: optional; uses `telegram_chat_id` and `telegram_base_url` to send media. If adding Telegram features, reuse `replace(base_url, telegram_base_url)` pattern.

Where to look in the repo
- `Frigate_Camera_Notifications/Beta.yaml` — primary, feature-rich blueprint (large templates, loops, debug output). Use as the canonical example.
- `Frigate_Camera_Notifications/Stable.yaml` — stable variant; compare when backporting changes.
- `Frigate_Camera_Notifications/README.md` — feature list and user-facing guidance.

If anything is unclear
- Ask for the intended runtime scenario (local HA install vs. supervised/OS) and test payload examples (a sample Frigate review JSON) if you need to run integration tests or simulate MQTT.

Targets for follow-up improvements (low-risk)
- Add a small example MQTT payload file under `tests/fixtures/` and a simple script to publish it for local testing.
- Add a `CONTRIBUTING.md` with a minimal validation checklist (yamllint, blueprint import, smoke test publish).

-- end --
