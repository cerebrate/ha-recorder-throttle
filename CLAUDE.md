# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Home Assistant custom integration (HACS) that throttles the recorder's database writes
per entity, driven by labels (`rec-off` / `rec-1min` / `rec-5min` / `rec-10min`). There is
no build step, no package manager, and no test suite — this is a pure Python/JS source tree
that HA loads directly from `custom_components/recorder_throttle/`. There is no local dev
loop for running it; changes are verified by reading the code, running the CI checks below,
and (for behavioral changes) manual smoke-testing against a real Home Assistant instance.

## Commands

- `python3 scripts/sync_card_version.py --check` — verify the card's console-banner version
  matches `manifest.json`'s `version`; CI gate (`validate.yml` → `card-version-sync`).
- `python3 scripts/sync_card_version.py --fix` — rewrite the banner to match. **Run this
  after every `manifest.json` version bump**, or CI fails.
- CI (`.github/workflows/validate.yml`) also runs `hassfest` (HA's own manifest/schema
  validator) and the `hacs/action` validator on every push/PR — there's no local equivalent,
  so keep `manifest.json`, `hacs.json`, and `strings.json`/`translations/*.json` internally
  consistent by hand.
- Releasing: push a `v*` tag; `.github/workflows/release.yml` extracts that version's
  section from `CHANGELOG.md` and publishes a GitHub Release (HACS updates from Releases,
  not bare tags). So a release requires, in order: bump `manifest.json` version → run
  `sync_card_version.py --fix` → add a `## <version>` section to `CHANGELOG.md` → tag.
  `manifest.json`'s `version` and the `CHANGELOG.md` heading are unprefixed (e.g. `1.0`);
  the git tag adds the `v` (e.g. `v1.0`) — the release workflow strips it back off to match.

## Architecture

Everything lives in `custom_components/recorder_throttle/`:

- **`__init__.py`** — all the runtime logic (setup, the recorder hook, services, the
  periodic scan, Lovelace card registration). See "The recorder hook" below.
- **`const.py`** — the label→interval table, label colors/icons, config-flow option keys
  and their defaults. Change throttle intervals or add a new policy level here first.
- **`config_flow.py`** — the settings dialog (config entry + options flow), single-instance.
  Fields are grouped into UI "sections" (`scan`, `auto`) and flattened back to the flat
  option keys `const.py`/`__init__.py` expect (`_flatten()`).
- **`repairs.py`** — the in-app "Fix" flow for the heavy-writers Repairs issue (throttle /
  accept / stop reporting). Recomputes the flagged entity list itself rather than trusting
  stale issue data, since time has passed since the issue was raised.
- **`recorder-throttle-card.js`** — the bundled Lovelace management card (vanilla JS custom
  element, no build step, no framework). Localized inline (`de`/`en`, mirrors HA's language).
  Its console-info version banner must stay in sync with `manifest.json` (see Commands).
- **`strings.json`** / **`translations/{en,de}.json`** — config-flow, options-flow, and
  Repairs UI text. Keep both languages in sync when adding user-facing strings. Note history:
  translation strings must not contain raw URLs (hassfest rejects that — see CHANGELOG 0.8.2).

### The recorder hook (the core mechanism)

The integration replaces the instance attribute
`Recorder._process_state_changed_event_into_session` on the live recorder instance with a
wrapper (`_install_hook` in `__init__.py`). The wrapper runs synchronously in the recorder
thread and, for each state-changed event:

1. If throttling is globally disabled, or the entity has no policy → pass through unchanged
   (call the original method). **No policy means "record normally", never "drop".**
2. If the entity's policy is `off` (interval 0) → drop (return `None`, no DB row).
3. If the entity is still inside its throttle interval → drop.
4. Otherwise → update the last-write timestamp and pass through.

Only cases 2 and 3 return `None`; case 1 (no explicit policy) is the overwhelming majority
of events and is always passed through. This is deliberately over-commented in the source
because reading the `return None` branches in isolation has previously led to the wrong
conclusion ("it discards everything") — don't re-introduce that ambiguity if you touch this
function.

This is a **fail-safe** design end-to-end:
- If the hook can't be installed (e.g. after an HA core update changes recorder internals),
  setup does not raise — it logs, raises a Repairs issue, and the recorder runs unthrottled.
- If the recorder is reloaded at runtime (common in development), the patched method gets
  replaced; a 30s interval (`_ensure_hook`) detects this (checks for the `_rt_wrapped`
  marker) and silently re-installs the hook.
- The live state machine, automations, and UI are never touched — only whether a `states`
  row gets written. Long-term statistics are computed upstream of this hook, so they survive
  throttling (including `off`) for entities that have a `state_class`.

Because this hooks a **private/internal** recorder method (`_process_state_changed_event_into_session`),
any change here needs manual verification against a real HA instance after HA core version
bumps: confirm a throttled sensor's writes actually drop, and that a broken hook fails open
rather than raising.

### Policy resolution

Policies are entirely derived from HA's label registry, not stored by this integration
directly: `_rebuild_policies` scans all entity registry entries for `rec-*` labels and
builds an `entity_id -> interval` dict. It's recomputed on registry/label update events and
after any `set_policy`/`set_accepted` service call — there's no incremental update path, the
whole map is rebuilt each time (cheap enough at HA's typical entity counts). An entity with
multiple `rec-*` labels resolves to the *most restrictive* one (`_more_restrictive`: `off`
beats any interval, and the larger interval wins between two intervals).

The global enabled/disabled switch (`recorder_throttle.set_enabled`) is intentionally
**not** stored in config-entry options — writing to options triggers the entry's update
listener, which reloads the whole integration (bad when you're urgently trying to switch
throttling off). It's persisted separately via `homeassistant.helpers.storage.Store`
(see `STORAGE_KEY`/`STORAGE_VERSION` in `const.py`) and restored on `async_setup_entry`.

### Services (`__init__.py`, registered in `_register_services`)

`set_policy`, `set_enabled`, `rebuild`, `top_writers` (returns response data — busiest
writers + running dropped/passed totals), `set_accepted`. Schemas are defined inline with
`voluptuous`; `services.yaml` documents them for the HA UI and must be kept in sync with the
schemas if parameters change.

### Card delivery

The card is served two ways, in order of preference, both idempotent and race-safe against
repeated calls: as a Lovelace storage-mode *resource* (`_register_lovelace_resource`,
preferred — survives reliably), falling back to `add_extra_js_url` if Lovelace is in YAML
mode or resources aren't available. `async_remove_entry` reverses the resource registration
on uninstall so removing the integration doesn't leave a dangling dashboard resource pointing
at a URL nothing serves anymore.

## Conventions

- Keep PRs small and focused (see CONTRIBUTING.md) — this is treated as a small, tightly
  scoped integration, not a platform.
- All code, comments, and log messages are in English; user-facing strings need both `en`
  and `de` translations.
- Broad `except Exception` blocks around registry/hook/registration code are intentional
  fail-safe boundaries (recorder must never stop recording because of a bug in this
  integration) — don't narrow them to "clean up" unless you're deliberately changing that
  guarantee, and if you do, re-verify the fail-open behavior manually.
- Never commit instance-specific data (tokens, internal hostnames/IPs, real entity/person
  names) — this repo's issues/PRs are public.
