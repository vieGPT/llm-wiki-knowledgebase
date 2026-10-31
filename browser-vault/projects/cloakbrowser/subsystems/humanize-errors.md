---
title: >-
    Humanize errors
category: subsystems
tags: [humanize, playwright, error-handling, api-compatibility]
aliases: [ActionabilityError, StealthDomError, humanize error hierarchy]
relationships:
  - target: "[[projects/cloakbrowser/subsystems/humanize-engine]]"
    type: related_to
sources:
  - ~/code/CloakBrowser/cloakbrowser/human/errors.py
  - ~/code/CloakBrowser/cloakbrowser/human/fields.py
  - ~/code/CloakBrowser/cloakbrowser/human/world.py
  - ~/code/CloakBrowser/cloakbrowser/human/engine.py
  - ~/code/CloakBrowser/dotnet/README.md
  - ~/code/CloakBrowser/js/src/human/errors.ts
  - ~/code/CloakBrowser/js/src/human/engine.ts
  - ~/code/CloakBrowser/tests/test_humanize_unit.py
summary: >-
    The two legacy humanize error hierarchies (ActionabilityError with 7 children, StealthDomError with 3), kept importable for compatibility while the engine raises Playwright errors.
provenance:
  extracted: 0.85
  inferred: 0.13
  ambiguous: 0.02
base_confidence: 0.54
lifecycle: draft
lifecycle_changed: 2026-10-31
tier: supporting
created: 2026-10-31T05:41:41+01:00
updated: 2026-10-31T10:47:25+01:00
---

# Humanize errors

`cloakbrowser/human/errors.py` defines two error hierarchies, both subclasses of `RuntimeError` and both exported from `cloakbrowser.human`. They are **kept for API compatibility**: the unified [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]] raises Playwright `Error` / `TimeoutError` (with a call log) instead, so existing `except` clauses and imports keep working without the engine ever raising these. The module docstring says the same set exists in the JS port (`js/src/human/errors.ts`) and the .NET port (`Human/Actionability.cs`); see [[projects/cloakbrowser/architecture/python-js-wrapper-parity|Python/JS wrapper parity]].

## ActionabilityError family

`ActionabilityError(selector, check, message)` is the base for actionability failures. It stores `selector` and `check` and formats "Element `<selector>` failed `<check>` check: `<message>`". Each child fixes the `check` name:

| Class | `check` | Meaning |
|---|---|---|
| `ElementNotAttachedError` | `attached` | element not found in DOM |
| `ElementNotVisibleError` | `visible` | element is not visible |
| `ElementNotStableError` | `stable` | element position is still changing |
| `ElementNotEnabledError` | `enabled` | element is disabled |
| `ElementNotEditableError` | `editable` | element is not editable |
| `ElementNotReceivingEventsError` | `pointer_events` | element is covered by another element; carries `covering_tag` |
| `ElementTargetChangedError` | `target_identity` | selector resolved to a different element before input dispatch |

The checks map onto the state sets the engine waits for (visible, enabled, editable, stable), its hit-target check, and its rule that a target replaced while the cursor travels is never pressed. In the current engine those conditions become retry reasons in a Playwright `TimeoutError` call log ("element is not visible", "`<div id=over>` intercepts pointer events") rather than typed exceptions. ^[inferred]

## StealthDomError family

`StealthDomError` is the base for isolated-world DOM helper failures. The source labels these as raised by the previous humanize layer; the engine no longer raises them.

| Class | Message |
|---|---|
| `UnsupportedHumanizeSelectorError` | humanized selector is not supported |
| `StealthWorldUnavailableError` | humanized DOM read requires an active isolated world |
| `StealthEvaluationError` | isolated-world DOM evaluation failed for a selector |

The first one became obsolete when the engine started running Playwright's own InjectedScript selector engine in the isolated world, which accepts the full selector syntax. ^[inferred]

## Errors the engine actually raises

| Error | Defined in | When |
|---|---|---|
| Playwright `Error` | Playwright | strict-mode violations, not fillable targets, malformed date/range values, `type=color`, element never focused, ElementHandle that shares its box with another element |
| Playwright `TimeoutError` | Playwright | deadline exceeded while retrying; also when a call stalls (see below) |
| `StealthWorldError` | `world.py` (subclass of Playwright `Error`) | an isolated-world evaluation failed; never falls back to the page |
| `StaleElement` | `world.py` (subclass of Playwright `Error`) | world-local element id gone; retried for locators, final for element handles |
| `PageStalled` | `world.py` (plain `Exception`) | a browser call got no answer within the action timeout (#619); deliberately not a Playwright `Error` so the engine's `except Error` fallbacks let it through, and the patch layer converts it to `TimeoutError` |
| `FieldKindError` | `fields.py` | non-numeric text for `input[type=number]`; wrapped into a Playwright `Error` by `fill` |
| `RawFallback` | `engine.py` | internal signal, not user-facing: run Playwright's own read or dispatch for a hidden or same-box ElementHandle |

Pinned by tests (B7): `tests/test_humanize_unit.py::test_error_classes_stay_importable` ("Same compatibility set as JS errors.ts / .NET Actionability.cs") checks the following:

- all seven `ElementNot…`/`ElementTargetChanged` classes subclass both `ActionabilityError` and `RuntimeError`;
- `ElementNotReceivingEventsError("#b", "div")` carries `check="pointer_events"` and the message "Element '#b' failed pointer_events check: element is covered by <div>";
- the three stealth classes subclass `StealthDomError`.

The real-browser suite shows what users actually get: Playwright `TimeoutError` with "intercepts pointer events", "outside of the viewport" or "waiting for the page to respond", and Playwright `Error` with "strict mode violation", "not attached", "shares its box" or "cannot be filled" (see [[projects/cloakbrowser/subsystems/humanize-engine|Humanize engine]]).

## .NET difference

The .NET client still throws the typed `ElementNot…Error` classes from its public standalone checks (`Actionability.EnsureActionableAsync`, `EnsureStableAsync`, `CheckPointerEventsAsync`), which run on the same engine. Humanized actions there throw `PlaywrightException` and `System.TimeoutException`, matching Python's Playwright errors. See [[projects/cloakbrowser/subsystems/dotnet-client|.NET client]].

## TypeScript port

`js/src/human/errors.ts` defines the same ten classes. They are kept "so existing `instanceof` imports keep compiling", and the unified engine throws Playwright-style `Error` / `TimeoutError` with a call log instead. At commit b1a00ed the only TS module that references these classes is `human/index.ts`, which re-exports them. `engine.ts` never imports or throws them, so the TS port matches Python: compatibility only.

Differences from the Python port:

- The base classes extend `Error`, not `RuntimeError`.
- The covering element's tag is a camelCase property, `coveringTag` (default `'unknown'`).
- `StealthDomError` sets `name` from `new.target.name`, so every subclass reports its own class name without overriding the constructor.

Engine-internal signals in TS are `Retry` (not exported) and the exported `RawFallback`. They mirror the Python control-flow exceptions.

## Open Questions

- Resolved for TypeScript (see above): compatibility only, and nothing in `js/src` throws these classes. The .NET client still throws them from its standalone checks.

## Class stubs (ast-extract)

- [[projects/cloakbrowser/subsystems/actionability-error|ActionabilityError]]
- [[projects/cloakbrowser/subsystems/element-not-attached-error|ElementNotAttachedError]]
- [[projects/cloakbrowser/subsystems/element-not-editable-error|ElementNotEditableError]]
- [[projects/cloakbrowser/subsystems/element-not-enabled-error|ElementNotEnabledError]]
- [[projects/cloakbrowser/subsystems/element-not-receiving-events-error|ElementNotReceivingEventsError]]
- [[projects/cloakbrowser/subsystems/element-not-stable-error|ElementNotStableError]]
- [[projects/cloakbrowser/subsystems/element-not-visible-error|ElementNotVisibleError]]
- [[projects/cloakbrowser/subsystems/element-target-changed-error|ElementTargetChangedError]]
- [[projects/cloakbrowser/subsystems/field-kind-error|FieldKindError]]
- [[projects/cloakbrowser/subsystems/page-stalled|PageStalled]]
- [[projects/cloakbrowser/subsystems/raw-fallback|RawFallback]]
- [[projects/cloakbrowser/subsystems/retry|Retry]]
- [[projects/cloakbrowser/subsystems/stale-element|StaleElement]]
- [[projects/cloakbrowser/subsystems/stealth-dom-error|StealthDomError]]
- [[projects/cloakbrowser/subsystems/stealth-evaluation-error|StealthEvaluationError]]
- [[projects/cloakbrowser/subsystems/stealth-world-error|StealthWorldError]]
- [[projects/cloakbrowser/subsystems/stealth-world-unavailable-error|StealthWorldUnavailableError]]
- [[projects/cloakbrowser/subsystems/unsupported-humanize-selector-error|UnsupportedHumanizeSelectorError]]

## Sources

Commit `b1a00ed` (26/10/2026):

- ~/code/CloakBrowser/cloakbrowser/human/errors.py
- ~/code/CloakBrowser/cloakbrowser/human/fields.py
- ~/code/CloakBrowser/cloakbrowser/human/world.py
- ~/code/CloakBrowser/cloakbrowser/human/engine.py
- ~/code/CloakBrowser/dotnet/README.md
- ~/code/CloakBrowser/js/src/human/errors.ts, js/src/human/engine.ts (commit b1a00ed, B4 batch)
- ~/code/CloakBrowser/tests/test_humanize_unit.py (commit b1a00ed, B7 batch)
