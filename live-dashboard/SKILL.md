---
name: live-dashboard
description: Build a live, multi-panel dashboard as a Claude Live Artifact - one that re-queries connectors on its own schedule, persists user state, and survives reloads. Use this any time someone asks for a "dashboard," "live artifact," "status page," "ops tracker," "executive overview," or any HTML/artifact that pulls from connectors and is supposed to keep showing fresh data over time, even if they don't say the word "dashboard." Especially use it before designing any UI for a connector-backed artifact - the most common failure mode is building the UI first and discovering at the end that the data shape was different than assumed.
---

# Live Dashboard

Live-artifact dashboards have a specific failure mode: the assistant designs the UI, plugs in MCP connector calls at the end, and then discovers the actual response shape doesn't match what the UI assumed. The dashboard ships looking polished and rendering nothing. This skill encodes the working pattern: data first, UI second, with explicit contracts for the things that bite you later (date awareness, cache state, refresh feedback, storage versioning, sandbox quirks).

The phases below are deliberately ordered. Skipping any of them to "save time" is exactly what causes the multi-day rebuild loop later.

## Phase 0 - Understand the request before assuming anything

Even when the user's prompt looks detailed, treat the first turn as a chance to *confirm* rather than assume. A "build me a dashboard for X" prompt almost always has unstated assumptions, and resolving them up front is much cheaper than rebuilding after the user sees the wrong thing.

Read the user's request carefully. Identify what's already specified (panel list, connectors, layout, brand, refresh cadence) and what's still ambiguous. Then use the **AskUserQuestion** tool to fill the gaps - aim for 2-3 focused multiple-choice questions, not a barrage. Don't re-ask things the user already answered.

Common gaps worth probing:

- **Audience and purpose.** Personal organizer, team ops tool, leadership rollup, or external/customer-facing? The same data renders very differently in each context - a "messy is fine" personal dashboard vs a "polished and trusted" exec view.
- **Refresh cadence.** Sub-minute (live monitoring), every few minutes (working dashboard), hourly (reports), daily (digest)? This determines auto-refresh rules, MCP rate-limit pressure, and whether you need streaming-like behavior.
- **Scope per panel.** "Show Linear issues" could mean current open, last N days, a specific project, a specific assignee. "Show Slack" could mean DMs only, channels you're in, channels matching a keyword. Clarify the slice - it's the single most common ambiguity.
- **Interactions.** Read-only display, or filters / drill-downs / manual entry (notes, ignore lists, reminders) / write-back to the source (reply to messages, close tickets, add comments)?
- **Date awareness.** Always "right now," or a date picker that scopes everything to a selected day/week/quarter? If the latter, what's the user's expectation for how past and future dates behave?
- **Definition of done.** What makes this dashboard succeed? "I can stop opening 5 tabs in the morning" is a very different success criterion than "leadership trusts the numbers without me presenting them."

If the user gives a screenshot to replicate, examine each section and ask about every data source you can't identify with confidence. Don't guess at sources - "the screenshot shows 47 bugs split four ways; where do those numbers come from and how is each bug tagged?" is much better than picking a plausible Linear label and finding out at the end it doesn't exist.

If the user pushes back ("just build it, I'll iterate"), respect that - but state your assumptions explicitly in your build response so they can correct any wrong ones before you've invested a lot in the wrong direction.

The goal of this phase isn't to slow things down. It's to make sure the next four phases (discovery, branding, contracts, build) are working toward the dashboard the user actually wants, not the one you guessed at.

## Phase 1 - Discovery before UI

Before writing a single line of HTML, call each connector you plan to use **once** and look at the actual response. MCP wrappers reshape the underlying API in ways that aren't documented externally:

- Slack tools often return markdown text wrapped in `{messages: "..."}` or `{results: "..."}` - sometimes as a parsed object, sometimes as a raw JSON string.
- Outlook returns JSON embedded inside content blocks; its date-range parameters aren't always honored.
- Linear returns mostly clean JSON but has cross-team noise on shared fetches.
- Notion may return raw page IDs in place of full URLs.

For each panel the user has asked for, do this:

1. Identify the connector tool that backs it (`outlook_calendar_search`, `list_issues`, etc.).
2. Call it once with realistic args. Inspect the response shape - array? object? content-block array? wrapped string?
3. Write down what you observed in a comment at the top of the eventual artifact so the next person editing it doesn't have to re-probe.
4. If a feature the user asked for cannot be backed by any available tool, **tell the user before building**. A common example: many email connectors are read-only - no reply or delete tools - so promised features have to be dropped or rephrased as "Open in Outlook" links. Never fake interactivity that the connector can't deliver.

Only after this discovery pass should you draft the UI. If you find yourself wanting to skip ahead because "the data is probably standard JSON," that's the signal that you'll be back here in two days rewriting parsers.

## Phase 1.5 - Branding decision (ask before writing CSS)

Once the data shapes are clear, but before writing any CSS, ask the user how they want the dashboard themed. Don't guess - the same dashboard wired the same way looks completely different in a clean neutral palette vs a heavily branded one, and the user may want either depending on context (internal tool vs customer-facing artifact vs personal use).

**Look for available brand/theme skills first.** At session start the `available_skills` list often includes one or more brand or theme skills - names usually end in `-brand`, `-theme`, or contain `theme` / `style`. Scan that list before asking so you can present the user with the option to apply what they already have installed.

**Then ask the user**, using the AskUserQuestion tool, something like:

> Should I apply a brand or theme?
> - **Apply [detected brand skill]** - use the rules from that skill (recommended if installed)
> - **Use a specific theme** - tell me the colors / fonts / vibe you want
> - **Neutral defaults** - a modern, unbranded dark theme; clean and portable

If the user picks the brand skill, **read its SKILL.md before writing any CSS** so colors, typography, and spacing come from the skill rather than ad-hoc choices. If they pick "a specific theme," capture their description as constants at the top of the artifact so it's easy to revisit. If they pick "neutral defaults," use:

- Page bg `#0F1419`, panel bg `#1A1F26`, border `#252C36` (1px)
- Text `#E6EDF3` primary, `#9FB4C7` secondary, `#6F8294` muted
- Accent `#00B0FF` (one cool color, not over-saturated)
- Modern system font stack
- 8px panel radius, 16px grid gap

These neutral defaults are explicitly *untheme* - they don't imitate any specific brand, so swapping in a real brand later is a small change rather than a rewrite.

Why this step matters: in past dashboard builds, brand decisions got baked in implicitly from whatever the user wrote in their initial prompt, without ever offering a clean alternative. Shareable skills should respect that the next user may want something completely different.

## Phase 2 - Write the contracts up front

Multi-panel dashboards accumulate complexity in four places, and that complexity will regress on you if it's spread across the codebase instead of pinned in one spot. Before writing the panel code, write these contracts as top-of-file constants and comments.

**Date awareness.** Decide once, for every panel, how it handles past, today, and future dates. The single most common bug across dashboard rewrites is one panel anchoring to today while another anchors to the date picker. The cleanest rule that holds up: every panel is date-aware; the date picker is the single source of truth; if a panel's underlying data source has no notion of past/future (current Slack channels, current open Linear queue), that panel renders current state regardless of picker and labels itself accordingly. Write that rule as a comment at the top of the script so future edits don't reinvent it per-panel.

**Cache state.** Don't use truthy-checks like `!!cGet(key)` to mean "has this date been loaded" - they answer false for any date with zero results, which makes the dashboard re-fetch every empty day forever. Use three explicit states: `never-loaded`, `loaded-empty`, `loaded-with-data`. A simple `cHasEntry(key)` helper that asks "was this key ever written?" disambiguates them and is one line of code.

**Refresh feedback.** Auto-refresh and manual refresh need different behavior. The N-minute tick should be silent when there's already data on screen (no flicker). The user clicking a panel's ↻ button is an explicit action and **must** show visible feedback (spinner, "fetching…" label, status dot change). Silent successful refreshes feel broken - the user clicked the button, something must happen on screen. Implement these as two distinct code paths.

**Storage versioning.** Anything persisted to `localStorage` (layout, ignored items, preferences) should have a version suffix on the key (`mydash_layout_v1`). When the schema changes, bump to v2 and add a one-time migration that preserves what's still valid and resets what isn't. Don't silently overwrite - users have manual choices saved there.

## Phase 3 - Build with environment awareness

A handful of platform quirks worth knowing up front rather than rediscovering mid-build.

**Grid layout doesn't constrain widths by default.** `grid-template-columns: repeat(N, 1fr)` divides *free* space, not total space. Any panel with content larger than its 1fr share will push its column past the others - and dashboards routinely have panels with user-generated text of unknown length (Slack messages, long ticket titles, Notion previews). The reliable form is `repeat(N, minmax(0, 1fr))` plus `min-width: 0` on the panel container. Without both, you'll ship with one column noticeably wider than the others and won't know why.

**Iframe sandbox capabilities vary.** Artifact runtimes change over time. Browser-platform APIs (`window.open`, `navigator.clipboard.writeText`, popup-style `target="_blank"`) may or may not work depending on current sandbox flags in the host. Two safe practices: before relying on any of these, do a quick runtime capability check inside the artifact; and for clipboard specifically, keep an `execCommand('copy')` from a hidden textarea as a fallback path, with a visible "select + ⌘C" affordance if both fail. Don't promise behavior that depends on an API you didn't verify in *this* environment.

**Error rendering, not silent spinners.** Every fetch path needs to terminate in either real data or a visible error block - never an indefinite spinner. When an MCP call fails, render the actual error text in the panel (red block, monospace) with a retry button. The first dashboard you build will have a connector that doesn't work in your environment, and the only way you'll know is if the error is on screen.

**Visible labeled lists, not dropdowns.** When a user is operating a dashboard, hiding data behind a dropdown defeats the point - they need to see everything at once and scan. Default to visible labeled lists / sections. Only collapse if the list is truly long and there's an explicit search affordance to compensate.

## Phase 4 - The pre-change protocol

Once a dashboard is in use, edits get dangerous fast. Date math, cache logic, and shared state get re-derived per panel; a "small fix" can break three other panels. Before any non-trivial edit, do these three things in order:

1. **List every caller** of the function you're about to change. Use Grep - don't rely on memory. Multi-panel dashboards typically have 3-5 callers of every helper.
2. **Predict the side effects** in writing, even if just to yourself in the response to the user. Walk through: "if I change X, then Y will…". This is the step that catches the "I'll fix days-off and accidentally break the PTO indicator on past dates" class of bug.
3. **Audit your change before pushing it.** Re-read the diff with the side-effects list in hand. If you can't tell whether a predicted side effect happened, that's the signal to add a log line or test - not to push and see.

This protocol exists because moving fast on a multi-panel dashboard regresses the same handful of bugs (timezone parsing, anchor mismatch, cache-state confusion) over and over. The friction of the protocol is much cheaper than the round-trip of "ship → user finds bug → re-investigate → fix → introduce next bug."

## Phase 5 - Verification

Before reporting the dashboard done:

1. Actually load it. Look at each panel. Confirm real data is rendering, not skeletons.
2. Click each interactive element - resize buttons, refresh buttons, modals, filters, date picker.
3. Trigger at least one error path (e.g. select a date with no data) and confirm the error block renders the real error message.
4. If the dashboard is supposed to refresh on a schedule, wait one tick and confirm the data updates.

Never report "the dashboard is ready" without doing this. The cost of one final verification pass is much smaller than the cost of the user opening it, seeing blank panels, and losing trust on first impression.

## Recurring traps worth knowing

Five bug classes that show up across dashboard builds:

**Timezone parsing.** `new Date('YYYY-MM-DD')` parses as **UTC midnight**, not local midnight. In time zones west of UTC this is hours of drift, and `Math.floor()`-style day counts come out off by one - a 70-day count becomes 80, the user notices, you spend an hour finding the bug. Always parse date-only strings as `new Date(dateStr + 'T00:00:00')` for local-midnight interpretation.

**Empty-array vs not-loaded.** Don't conflate them. `cGet(key)` returning null for empty arrays + `!!cGet(key)` as the "is loaded" check = an infinite re-fetch loop for any date with zero results. Use a separate `cHasEntry(key)` helper.

**Naming drift on rename.** When a panel gets renamed (say, an "old name" → "new name" change requested mid-project), the rename needs to touch *user-visible* strings (titles, tooltips, log labels) but **not** code internals (state keys, function names, localStorage keys, DOM IDs). Touching internals corrupts saved state for every user. Split the rename into two passes from the start: labels vs identifiers, and keep the identifiers stable across the lifetime of the dashboard.

**Manual refresh suppression.** If the auto-tick has logic like "don't show spinner when cached data is on screen," that suppression must not extend to the manual refresh path. The user clicking ↻ is explicit; the spinner must show even when there's stale data already rendered.

**Grid column drift.** Covered above but worth repeating because it surfaces visually and looks like a layout bug rather than a CSS spec gotcha. `repeat(N, 1fr)` is almost never what you want for content-bearing dashboards - use `minmax(0, 1fr)`.

**`display: none` on a grid child reflows its siblings.** When a row uses a fixed-column track (e.g. `grid-template-columns: 6px 1fr auto` for a status-dot + content + meta pattern), conditionally hiding the first child with `display: none` removes it from the grid entirely. The remaining children slide left to fill columns starting from column 1 - so the "content" element ends up squeezed into the 6px dot column and the row looks blank. Use `visibility: hidden` or just `background: transparent` on the conditional element so the grid slot stays occupied. This bites whenever a row has a conditional indicator (unread dot, error icon, drag handle) that should disappear on certain states.

## Anti-patterns to avoid

- Designing the UI before probing the connectors.
- Pretending a feature works when the connector can't back it.
- Truthy-check empty-array caches.
- One panel that auto-refreshes a past date when every other panel froze it.
- Hiding critical actions behind dropdowns.
- Silent successful refreshes on user-clicked buttons.
- Rapid-fire edits without listing callers first.
- Reporting "done" without loading the artifact yourself.

## Working with the user

A few interaction patterns that hold up well on dashboard work:

When the user hands you a long spec - a numbered list, a paragraph of requirements, a screenshot annotated with arrows, whatever shape it takes - treat each requirement as a contract, and call out the ones you can't fulfill *before* you start building rather than silently dropping them. The user would rather hear "Zoom has no create-meeting tool, I'll open zoom.us/start instead" up front than discover it at delivery.

When the user reports a bug ("it's wrong" / "still broken"), don't jump to a fix. Ask one clarifying question if needed to make sure you're solving the right thing - selected date? what they expected? - and *then* trace the root cause through the codebase before editing. The cost of one clarifying question is much smaller than the cost of three wrong fixes.

When you're about to make a change you've made before in a different shape - date math, cache state, refresh logic - actively pause and check whether the existing pattern in the codebase already handles it. The skill exists in large part because these areas regress; treat them like load-bearing structure, not as code to rewrite ad-hoc.
