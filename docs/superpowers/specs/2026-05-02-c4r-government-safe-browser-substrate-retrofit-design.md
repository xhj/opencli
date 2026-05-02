# C4R Government-Safe Browser Substrate Retrofit Design

Date: 2026-05-02
Status: Downstream handoff from Crawl4Research
Scope: OpenCLI fork retrofit only, not C4R Python runtime refactor

Source spec: `/Users/xhj/Developer/Crawl4Research/docs/superpowers/specs/2026-05-02-opencli-government-safe-substrate-retrofit-design.md`

## 1. Executive Summary

This document records the OpenCLI-side browser substrate contract required by Crawl4Research (C4R) government-site acquisition.

C4R's user-facing goal is to let an individual researcher use a browser automation substrate to legally acquire public content that the researcher could manually access in a browser, while retaining auditable evidence of what happened. The browser substrate must therefore be close enough to legitimate manual browsing to answer a concrete evidence question: did this access run use an isolated dedicated profile, wait for an explicit readiness signal, avoid polluting page main world, capture HTTP evidence from tool-level channels, and recover only target-scoped state?

The current OpenCLI codebase already has useful pieces: Browser Bridge / daemon / extension, CDP or extension network capture, tab operations, cookie reading, selector/text/xhr waits, Markdown extraction, Readability, and Turndown. Those pieces do not yet form the government-safe substrate C4R needs:

- `workspace` is a logical session/window/cache namespace, not a physical browser profile.
- ready waits do not form a `page_ready_signal` evidence contract.
- extraction currently relies on main-world `evaluate` / `Function` paths.
- network capture can fall back to page-context `fetch` / `XMLHttpRequest` wrappers when session-level capture is unavailable.
- cookies can be read but not target-scope cleared with before/after evidence.
- origin storage has no fact/reset primitive.
- tab new/close is not equivalent to a recovery-grade fresh BrowserContext/fresh page primitive with evidence.

OpenCLI either gains this contract as a downstream C4R patch line, or C4R must replace OpenCLI with a different external tool that can satisfy the same contract.

## 2. Non-Goals

- Do not bypass login, CAPTCHA, paywalls, permission walls, or non-public access controls.
- Do not introduce proxy rotation, fingerprint rotation, automatic user-agent switching, automatic headless switching, or throughput-oriented crawling strategy.
- Do not implement challenge classification, manual-session classification, confidence scoring, or next-action policy inside OpenCLI.
- Do not convert page text, script counts, storage keys, or HTML shape evidence into challenge attribution labels.
- Do not clear the researcher's daily browser profile.
- Do not clear the whole dedicated browser profile.
- Do not clear unrelated-site cookies or storage.
- Do not move C4R site adapters into OpenCLI. Site-specific government adapters remain in the C4R-owned `opencli_plugins/critical-sites/` repository surface.

## 3. Required Contract

OpenCLI must expose a machine-readable capability surface for the C4R government-safe browser contract:

```json
{
  "government_safe_browser_contract": "c4r.opencli-gov-browser.v1",
  "features": {
    "browser_profile_id": true,
    "page_ready_signal": true,
    "extraction_isolation": ["isolated_world", "post_challenge_only"],
    "network_capture_layer": ["cdp_network_domain", "playwright_request_event", "har_export"],
    "scoped_cookie_clear": true,
    "origin_storage_fact": true,
    "origin_storage_reset": true,
    "fresh_context": true
  }
}
```

C4R will check the version and feature presence. C4R Python runtime must not inspect Chrome profile internals or probe whether isolation is "good enough" at runtime.

## 4. Dedicated Persistent Browser Profile

OpenCLI must support a manifest-driven `browser_profile_id` and map it to a dedicated persistent browser profile.

Requirements:

- The dedicated profile is physically isolated from the researcher's daily browser profile.
- Default execution is headful and uses a real Chrome binary.
- Normal session state persists across government-site visits for the same dedicated profile.
- `workspace` remains only a logical session/window/cache namespace and must not be treated as a profile substitute.
- Headless mode, Chromium binary selection, and user-agent changes are allowed only when explicitly declared by the C4R manifest.
- Evidence must return the actual `browser_profile_id`.
- C4R Python must not directly read or write Chrome profile files.

OpenCLI may choose its own profile registry file location and user-data-dir management strategy, as long as the above constraints hold.

## 5. Page Ready Signal

OpenCLI must support one closed, single-condition ready signal:

- `selector:<css-selector>`
- `cookie:<cookie-name>`
- `event:<event-name>`

Requirements:

- Do not accept OR, AND, weighted, or compound conditions.
- Return `page_ready_signal`, `page_ready_signal_hit`, and `page_ready_wait_ms`.
- If timeout occurs before the signal is hit, do not run body extraction. Return evidence and a non-`ok` status.
- `cookie:` ready must observe HttpOnly cookie name presence without recording cookie values.

## 6. Extraction Isolation

Government acquire must support:

- `isolated_world`: use a CDP isolated world or equivalent isolated execution context so extraction does not inject scripts into page main world.
- `post_challenge_only`: after ready signal hits, read an HTML snapshot and run Readability/Markdown conversion host-side or tool-side outside page main world.
- `none`: allowed only for explicitly declared non-challenge sites.

Government-site paths must not inject scripts into main world before the ready signal. Sites with known challenge behavior default to `isolated_world` or `post_challenge_only`. Evidence must return actual `extraction_isolation` and `extraction_source`.

If OpenCLI keeps using Readability/Turndown, those components must move into the isolated-world or post-challenge snapshot pipeline for government acquire. Main-world `Function` / `eval` must not remain the government acquire main path.

## 7. Network Capture Layer

Government detail acquire must declare and return the actual tool-level capture channel:

- `cdp_network_domain`
- `playwright_request_event`
- `har_export`

Government-site paths must not use page-context network wrapping:

- Do not wrap `window.fetch`.
- Do not wrap `XMLHttpRequest.prototype`.
- Do not inject network interception or rewrite scripts into main world.
- Do not silently fall back to JS wrappers when the requested capture layer is unavailable.

If the declared layer is unavailable, return structured `execution_error` evidence. `network_capture_layer = none` is allowed only for non-acquire actions that do not need HTTP evidence.

## 8. HTTP Evidence And Exit Semantics

OpenCLI must distinguish site HTTP errors from tool execution failures:

- If browser execution succeeds and observes `http_status >= 400`, OpenCLI should exit 0 and return JSON evidence with `status: "http_error"` and `http_status`.
- OpenCLI exceptions, missing profile, unavailable capture layer, contract mismatch, non-JSON output, and similar tool failures should use non-zero exit or `status: "execution_error"`.

This lets C4R preserve 400 / 403 / 412 as browser evidence instead of losing them as stderr-only process failures.

## 9. Scoped Recovery

OpenCLI must support recovery prelude primitives:

- `clear_target_cookies`
- `record_target_origin_storage`
- optional `clear_target_origin_storage`
- `fresh_browser_context`
- `bounded_pacing`
- `wait_for_page_ready`

Scope rules:

- Default cookie/storage scope is the exact target origin/host.
- Parent-domain or same-site subdomain scopes may only come from explicit frozen C4R manifest scope.
- Evidence must record each scope source as `derived` or `manifest`.
- Do not clear unrelated-site cookies/storage.
- Do not clear the daily browser profile.
- Storage facts record only key/database counts, not values.
- Origin storage reset only runs when the frozen profile explicitly permits it.

After scoped clear, recovery must start the next navigation from a fresh context/fresh page and must not rely on old page reload. Evidence must record `fresh_context: true`, old page closure, new page/context id, and a testable fact proving the old page was not reused.

If the current extension architecture cannot provide a real BrowserContext, OpenCLI must define an auditable equivalent isolation primitive. If that equivalent cannot be made valid, this is a blocker for OpenCLI as C4R's browser substrate.

## 10. Bounded Pacing

OpenCLI must support frozen wait profiles:

- waits before/after seed actions
- waits before/after detail opening
- DOM settle budget
- ready wait budget
- bounded jitter min/max

Actual sampled waits must be included in evidence. OpenCLI must not emit a human-likeness score, adapt waits dynamically, or retry indefinitely.

## 11. Minimum Acquire Evidence Envelope

Government acquire returns at least:

```json
{
  "status": "ok | empty | http_error | execution_error",
  "http_status": 412,
  "final_url": "...",
  "body_html": "...",
  "body_markdown": "...",
  "body_text": "...",
  "title": "...",
  "published_at": "...",
  "attachments": [],
  "evidence": {
    "browser_profile_id": "c4r-gov-zh-default-v1",
    "network_capture_layer": "cdp_network_domain",
    "page_ready_signal": "selector:.article-body",
    "page_ready_signal_hit": false,
    "page_ready_wait_ms": 10000,
    "extraction_isolation": "post_challenge_only",
    "extraction_source": "opencli_readability",
    "fresh_context": true,
    "cookie_scope": [{ "scope": "https://jyt.gansu.gov.cn", "source": "derived" }],
    "cookies_before": 3,
    "cookies_cleared": 3,
    "origin_storage_scope": [{ "scope": "https://jyt.gansu.gov.cn", "source": "derived" }],
    "origin_storage_before": { "localStorage_keys": 1, "sessionStorage_keys": 0, "indexedDB_databases": 0 },
    "origin_storage_after": { "localStorage_keys": 0, "sessionStorage_keys": 0, "indexedDB_databases": 0 },
    "origin_storage_reset": true,
    "wait_profile_id": "gansu-education-detail-recovery-v1",
    "pacing_events": [
      { "phase": "after_cookie_clear", "wait_ms": 1840 }
    ],
    "html_shape_signals": {
      "has_body": true,
      "script_count": 8,
      "content_element_count": 0,
      "visible_text_chars": 12
    },
    "stderr_tail": "..."
  }
}
```

`html_shape_signals` may contain only raw counts and booleans. Do not name them with attribution labels such as `challenge_script_only`.

## 12. Authority Placement

- C4R runtime: reads OpenCLI evidence and frozen site manifests.
- C4R/upstream agent: interprets evidence as access challenge, manual-session need, access-boundary hit, or next research action.
- C4R harness: owns site evidence collection, frozen manifest fields, smoke cases, fixtures, and contrast evidence.
- OpenCLI: owns government-safe browser substrate, profile carrier, network capture, ready wait, isolated/post-challenge extraction, cookie/storage/fresh-context primitives, and bounded pacing.
- Explicitly refused inside OpenCLI: challenge classifier, manual-session classifier, confidence scorer, next-action decider, proxy/fingerprint rotation, automatic bypass strategy, and adaptive human-likeness strategy.

OpenCLI may execute an explicit profile/ready/recovery contract, but must not convert evidence into challenge attribution or silently downgrade capture/isolation behavior.

## 13. Acceptance Criteria

Profile/session:

- OpenCLI supports manifest-driven `browser_profile_id`.
- Tests prove `workspace` is not equivalent to profile.
- Dedicated profile is physically isolated from daily profile.
- Default is headful real Chrome; headless/UA/binary changes are explicit.
- Evidence returns the requested actual `browser_profile_id`.
- Silent fallback to a different profile is reported as `execution_error`.
- Acceptance fixtures include contrast evidence: vanilla headless/main-world/no-ready failure versus frozen profile/ready/isolated-or-post-challenge success for the same site/query/manifest when available.

Ready/extraction:

- Supports `selector:`, `cookie:`, and `event:` single-condition ready signals.
- Evidence records hit/timeout/wait duration.
- Ready timeout prevents body extraction and prevents `ok`.
- `isolated_world` does not inject into main world.
- `post_challenge_only` snapshots only after ready hit and converts outside main world.
- Government acquire does not use main-world `Function` / `eval` as the main extraction path.

Network:

- Supports `cdp_network_domain` or an equivalent tool-level capture channel.
- Unavailable capture layer returns structured `execution_error`.
- Government acquire has no `fetch` / `XMLHttpRequest` wrapper fallback.
- Tests prove page context does not contain OpenCLI-injected network wrappers.

Recovery:

- Scoped cookie clear only affects exact target origin/host plus explicit manifest scopes.
- Origin storage fact records only counts.
- Origin storage reset requires explicit profile declaration.
- Recovery uses fresh context/fresh page, not old-page reload.
- Evidence records scope source, before/after facts, cleared count, fresh context, and pacing events.

Exit/evidence:

- HTTP 400 / 403 / 412 return JSON `http_error` evidence rather than tool failure.
- Tool failure returns `execution_error` or non-zero exit.
- JSON envelope schema has focused tests.

## 14. Suggested Implementation Sequence

1. Add government-safe capability version command and schema tests.
2. Implement `browser_profile_id` registry and dedicated persistent profile carrier.
3. Refactor government network capture to fail closed when tool-level capture is unavailable.
4. Add `page_ready_signal` primitive with selector/cookie/event evidence.
5. Implement `isolated_world` or `post_challenge_only` extraction.
6. Add scoped cookie clear and origin storage fact/reset primitives.
7. Add fresh context/fresh page recovery primitive.
8. Add bounded pacing/wait profile and sampled wait evidence.
9. Emit government acquire evidence envelope and add focused unit/integration tests.

Prefer local fixtures and mock browser substrate tests for daily verification. Real government-site checks are supplementary acceptance evidence, not a substitute for deterministic tests.
