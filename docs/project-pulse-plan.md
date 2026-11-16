# Project Pulse Dashboard: Implementation Plan

> Planning document for Mona's **Project Pulse** dashboard. The Planner wrote it for the Orchestrator, which will split it into phases and hand them to the **Designer** and the **Coder**.
>
> **Git control:** No agent stages, commits or pushes changes. The learner controls all git operations through Copilot CLI prompts.

---

## Summary

Project Pulse is a small static dashboard with no build tools. It lets Mona's contributors see at a glance:

- which projects are active
- who owns each one
- its status
- recent activity
- its priority or risk level
- a short summary

The app is made of three files in `app/`, served with `python3 -m http.server 5500`. A VS Code launch configuration named **Run Project Pulse Dashboard** starts the server and opens the dashboard.

| File | Owner | One-line role |
| --- | --- | --- |
| `app/index.html` | **Coder** | Semantic page shell plus an inline `<script>` that fetches `project-data.json` and renders `.project-card` elements |
| `app/styles.css` | **Designer** | All visual styling: the `.dashboard` layout, `.project-card` cards, status badges, priority treatment, responsive and accessible styles |
| `app/project-data.json` | **Coder** | The data, as a top-level `projects` array |
| `.vscode/launch.json` | **Coder** | Strict-JSON launch configuration that serves `app/` and opens `index.html` |

**Ownership decision:** The Coder owns `app/index.html` and `app/project-data.json`, because the HTML holds the fetch and render logic and the JSON is the data contract that logic uses. The Designer owns only `app/styles.css`. The Designer shapes the markup through the **class and attribute contract** below, not by editing `index.html`. This means no file has two owners, so the Designer and Coder can work in parallel.

Repository facts that shaped this plan:

- `app/` is empty.
- `.vscode/` contains only `tasks.json`.
- The devcontainer has no Python VS Code extension, so the launch config uses the built-in `node-terminal` debug type (part of VS Code's bundled JavaScript debugger) rather than `debugpy`.
- The Step 3 workflow checks for these literal strings:
  - in `index.html`: `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority`
  - in `styles.css`: `.dashboard`, `.project-card`, `border-radius`, `box-shadow`
  - in the JSON: `projects`, `name`, `owner`, `status`, `recentActivity`, `priority`
  - in `launch.json`: `Run Project Pulse Dashboard`, `index.html`
  - plus `python3 -m json.tool` must succeed on both JSON files

---

## Shared contract (fixed now so phases can run in parallel)

Both specialists must follow this contract exactly. Changing it means going back to the Orchestrator (see Dependencies).

### Data schema (`app/project-data.json`)

```text
{
  "projects": [
    {
      "name": string,           // required
      "owner": string,          // required
      "status": string,         // required; one of: "On Track", "At Risk", "Blocked", "Completed"
      "recentActivity": string, // required; short sentence
      "priority": string,       // required; one of: "High", "Medium", "Low"
      "summary": string         // optional; short contributor-friendly description
    }
  ]
}
```

### Rendered DOM structure (built by the Coder in `index.html`; styled by the Designer)

```text
body
└─ main.dashboard                          (landmark; aria-labelledby="dashboard-title")
   ├─ header.dashboard-header
   │  ├─ h1#dashboard-title                "Project Pulse"
   │  ├─ p.dashboard-subtitle              short tagline
   │  └─ ul.dashboard-summary              counts per status (li.summary-item > span.summary-count + span.summary-label)
   ├─ p.dashboard-message#dashboard-message   role="status" aria-live="polite"; loading/empty/error text
   │                                          modifier classes: .is-loading | .is-empty | .is-error ; [hidden] when not used
   └─ ul.project-grid                      role="list"
      └─ li > article.project-card         data-status="<slug>" data-priority="<slug>"
         ├─ header.project-card__header
         │  ├─ h2.project-card__name
         │  └─ span.status-badge.status-badge--<slug>          text = status value (never color alone)
         ├─ p.project-card__owner          "Owner: " + owner
         ├─ p.project-card__summary        (omitted if no summary)
         ├─ div.project-card__activity     dt/dd-style label "Recent activity" + text from recentActivity
         └─ span.priority-tag.priority-tag--<slug>             text = "Priority: High" etc.
```

**Slug rule** (the Coder implements it, the Designer relies on it): trim, lowercase, and replace runs of non-alphanumeric characters with `-`. Examples:

- `"On Track"` → `on-track`
- `"At Risk"` → `at-risk`
- `"Blocked"` → `blocked`
- `"Completed"` → `completed`
- `"High"` / `"Medium"` / `"Low"` → `high` / `medium` / `low`

Any unrecognized or missing value becomes the slug `unknown`.

---

## Ordered implementation steps

| # | Phase | Step | Owner | Files |
| --- | --- | --- | --- | --- |
| 0 | Plan | Save this plan as `docs/project-pulse-plan.md` | Orchestrator | `docs/project-pulse-plan.md` |
| 1a | Build (parallel) | Write the sample data that follows the schema | Coder | `app/project-data.json` |
| 1b | Build (parallel) | Write the page shell, fetch logic, rendering, and loading/empty/error states | Coder | `app/index.html` |
| 1c | Build (parallel) | Write the launch configuration | Coder | `.vscode/launch.json` |
| 1d | Build (parallel) | Write all visual styling for the contract classes | Designer | `app/styles.css` |
| 2 | Integrate (sequential) | Serve the app and check that the rendered markup matches the CSS hooks. The Designer reviews visually and accessibility. Fixes go to the file's owner only. | Orchestrator → Designer (review) → Coder / Designer (fixes in own files) | any of the above, by owner only |
| 3 | Validate (sequential) | Run all checks in **Validation expectations** and report the results | Orchestrator (with the Coder running commands) | none (read-only) |

Steps 1a, 1b and 1c are all Coder work and can be done in that order within one Coder task. Step 1d runs at the same time as the Coder's work.

---

## File assignments

| File | Owner | Purpose | Exact contents expected |
| --- | --- | --- | --- |
| `app/index.html` | **Coder** | Page structure and all runtime logic (fetch, validate, render, states) | See below |
| `app/styles.css` | **Designer** | Polished, responsive, accessible look | See below |
| `app/project-data.json` | **Coder** | Single source of project data | See below |
| `.vscode/launch.json` | **Coder** | One-click run from VS Code Run and Debug | See below |
| `docs/project-pulse-plan.md` | Orchestrator | This plan, saved verbatim | This document |

### `app/index.html` (Coder)

- `<!DOCTYPE html>`, `<html lang="en">`, `<meta charset="utf-8">`, and a viewport meta tag.
- `<title>Project Pulse</title>`, with the exact text **Project Pulse** also in the `<h1>`.
- `<link rel="stylesheet" href="styles.css">`, as a relative path.
- The static shell from the contract: `main.dashboard`, the header, the `#dashboard-message` live region (starts with "Loading projects…" and class `is-loading`), and an empty `ul.project-grid`.
- One inline `<script>` at the end of `body`, with no external JS file and no frameworks. It:
  - calls `fetch("project-data.json")` with a relative URL and checks `response.ok`
  - parses the JSON, checks that `data.projects` is an array, and normalizes each project
  - builds each card with `document.createElement` and `textContent`, **never** `innerHTML`, so data can't inject markup
  - gives each card the `project-card` class and `data-status` / `data-priority` attributes
  - reads `project.status`, `project.recentActivity` and `project.priority` as written. These literal names must appear in the file for the workflow checks.
  - fills in the per-status summary counts
  - handles the empty, error and missing-field cases listed under Edge cases
- No inline `style=` attributes and no `<style>` block. All visual styling belongs to the Designer.

### `app/styles.css` (Designer)

- Custom properties on `:root` for colors, spacing, radius and shadow.
- `.dashboard`: centered max-width container with page padding.
- `.dashboard-header`, `.dashboard-subtitle`, `.dashboard-summary`, `.summary-item`, `.summary-count`, `.summary-label`.
- `.dashboard-message`, plus `.is-loading`, `.is-empty` and `.is-error` variants. The error variant must be visually distinct but still readable.
- `.project-grid`: CSS Grid using `repeat(auto-fill, minmax(18rem, 1fr))`, with a gap and no list bullets or padding.
- `.project-card`: background, `border-radius`, `box-shadow`, padding, a flex column, and a hover/focus-within lift. The lift is turned off under `prefers-reduced-motion`.
- `.project-card__header`, `__name` (with `overflow-wrap: anywhere` for long names), `__owner`, `__summary`, `__activity`.
- `.status-badge` plus modifiers `--on-track`, `--at-risk`, `--blocked`, `--completed` and `--unknown`. Pill shape, with text contrast of at least 4.5:1.
- `.priority-tag` plus `--high`, `--medium`, `--low` and `--unknown`. Optional card accent through `.project-card[data-priority="high"]`, for example a left border.
- Responsive: one column on narrow screens (≤ 480px) with reduced padding, and the summary row wraps.
- Accessibility: a visible `:focus-visible` outline, no meaning shown by color alone (the badges contain text), and readable base font size and line-height.
- Optional: `@media (prefers-color-scheme: dark)` overrides via custom properties.

### `app/project-data.json` (Coder)

- Strict JSON: no comments and no trailing commas.
- A top-level `projects` key holding **6 projects**. Every one has `name`, `owner`, `status`, `recentActivity` and `priority`; most also have `summary`.
- Cover every status (`On Track`, `At Risk`, `Blocked`, `Completed`) and every priority (`High`, `Medium`, `Low`).
- Include one deliberately long project name to check text wrapping.
- Suggested entries (team-themed, fictional):
  - "Octodex Refresh"
  - "Contributor Onboarding Guide"
  - "CI Pipeline Speedup"
  - "Accessibility Audit"
  - "Docs Search"
  - a long name such as "Community Discussions Migration and Moderation Tooling Overhaul"

### `.vscode/launch.json` (Coder)

Strict JSON with no comments. Expected contents:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Project Pulse Dashboard",
      "type": "node-terminal",
      "request": "launch",
      "command": "python3 -m http.server 5500",
      "cwd": "${workspaceFolder}/app",
      "serverReadyAction": {
        "action": "openExternally",
        "pattern": "port ([0-9]+)",
        "uriFormat": "http://localhost:%s/index.html"
      }
    }
  ]
}
```

Why these values:

- `cwd` serves only `app/`, so `/` is the app and not the repository root.
- `uriFormat` opens `index.html` directly, so no directory listing appears.
- `pattern` matches Python's startup line, "Serving HTTP on … port 5500 (…)". This works whether Python binds to `0.0.0.0` or `::`.
- In Codespaces, `localhost:5500` is forwarded automatically.

---

## Designer responsibilities

1. Own and write **only** `app/styles.css`.
2. Build a polished dashboard, not a bare page. The first screen should clearly read as "Project Pulse": a strong title, a summary strip, and a card grid.
3. Include the required hooks: `.dashboard` and `.project-card`, plus every class or modifier in the contract, so no rendered element is unstyled.
4. Status badges: a distinct, accessible color for each status, with the text label always visible.
5. Priority treatment: a clear visual hierarchy (High stands out most, Low is quiet), using both the tag and an optional card accent.
6. Use `border-radius`, `box-shadow`, clear typography and readable spacing.
7. Responsive layout from about 320px up to wide desktop screens.
8. Accessibility:
   - contrast of at least 4.5:1 for text and at least 3:1 for UI boundaries
   - `:focus-visible` styles
   - support for `prefers-reduced-motion`
   - no information shown by color alone
9. In Phase 2, review the rendered page and report any markup changes needed (for example, a missing wrapper) **to the Orchestrator**, who assigns them to the Coder. Do not edit `index.html`.
10. Report design decisions, tradeoffs (for example, grid vs. flex, or why the colors were chosen), the files touched, and validation recommendations.

## Coder responsibilities

1. Own and write `app/index.html`, `app/project-data.json` and `.vscode/launch.json`.
2. Follow the contract's DOM structure, class names, `data-*` attributes and slug rule exactly.
3. Keep the logic explicit and deterministic: one `loadProjects()`, one `renderProjects(projects)`, one `renderMessage(kind, text)`, and a small `toSlug(value)`. Use descriptive names and no clever abstractions.
4. Make errors explicit and helpful (see Edge cases). Log technical details with `console.error` and show friendly text in `#dashboard-message`.
5. Add no styling in HTML. Do not edit `app/styles.css`.
6. Write `.vscode/launch.json` exactly as specified: strict JSON, `cwd` set to `${workspaceFolder}/app`, opening `index.html`.
7. Run the command-line validation checks before reporting, and report what changed, what was validated, and any remaining risks.

---

## Dependencies

| Dependency | Type | How it's resolved |
| --- | --- | --- |
| `index.html` rendering ↔ `project-data.json` schema | Data | The schema is fixed in this plan, so both are Coder-owned and written by the same agent |
| `styles.css` selectors ↔ `index.html` class names | Interface | The class/attribute contract is fixed in this plan, so they can be built in parallel |
| `styles.css` badge/priority modifiers ↔ data values and slug rule | Data | Allowed values and the slug rule are fixed in this plan |
| `launch.json` ↔ `app/index.html` existing at `app/` | Runtime | Only matters for the Phase 2/3 run check, not for writing the file |
| Phase 2 integration review ↔ all of Phase 1 | Sequential | Can't start until all four files exist |
| Phase 3 validation ↔ Phase 2 fixes | Sequential | Validate the final state only |
| Any contract change | Blocking | Must go through the Orchestrator, which updates both specialists before either edits again |

## Parallel work decisions

**Can run in parallel (Phase 1):**

- **Designer → `app/styles.css`** and **Coder → `app/index.html`, `app/project-data.json`, `.vscode/launch.json`**.
- Why this is safe: the file scopes don't overlap (Designer has one file, Coder has three, none shared). The only link between them, class names and data values, is fixed in the shared contract above, so neither agent needs the other's output to start.

**Must run sequentially:**

- **Inside the Coder's task:** `project-data.json`, then `index.html`, then `launch.json`. One agent owns all three, so running them in parallel gains nothing, and writing the data first lets the Coder test rendering right away.
- **Phase 2 (integration) after Phase 1:** it needs both the rendered markup and the CSS to exist.
- **Phase 2 fixes:** any fix to `index.html` goes to the Coder and any fix to `styles.css` goes to the Designer. If both files need changes at once, they can run in parallel only if the contract is unchanged. A contract change makes them sequential: update the contract first, then the Coder, then the Designer.
- **Phase 3 (validation) after all fixes.**
- **Never:** two agents editing the same file in the same phase.

---

## Edge cases

| Case | Expected behavior | Owner |
| --- | --- | --- |
| Opened via `file://` (double-clicking `index.html`) | `fetch` fails (CORS/blocked). Show an error in `#dashboard-message` (`.is-error`): "Couldn't load project data. Open the dashboard with the **Run Project Pulse Dashboard** launch configuration (or run `python3 -m http.server 5500` in `app/`)." Hide the empty grid. | Coder (logic), Designer (error style) |
| HTTP error (404/500) | Check `response.ok`; show an error that includes the status code; `console.error` the details | Coder |
| Invalid JSON | Catch the parse error and show an error message saying the data file is malformed | Coder |
| `projects` missing or not an array | Show an error that names the missing or invalid key | Coder |
| `projects: []` | Show the `.is-empty` message "No projects yet. Add entries to project-data.json." and set summary counts to 0 | Coder, Designer |
| Missing or empty fields on a project | Still render the card with fallbacks: name "Untitled project", owner "Unassigned", status "Unknown" (slug `unknown`), recentActivity "No recent activity", priority "Unknown". Skip `summary` if it's absent. Skip entries that aren't objects and `console.warn` | Coder |
| Unknown status or priority values (for example "Paused") | Show the original text; slug becomes `unknown` unless it matches a known slug; the neutral badge style applies | Coder, Designer |
| Long names, owners or activity text | Wrap with `overflow-wrap: anywhere`; no horizontal scroll; cards in the same row stay aligned (grid `align-items: stretch`) | Designer |
| Many projects | The grid flows naturally; no fixed heights | Designer |
| Non-string values (numbers, null) | Convert with `String()`; treat null or undefined as missing | Coder |
| HTML in data values | Shown as plain text because the Coder uses `textContent` | Coder |
| Port 5500 already in use | The server fails to start and serverReadyAction never fires. Document: stop the other process or the previous run. Leave the port fixed (the Step 3 prompt requires 5500). | Coder (report) |
| Browser not opening in Codespaces | Open the forwarded port 5500 from the Ports panel and go to `/index.html` | Coder (report) |
| Reduced motion / keyboard users | No animation under `prefers-reduced-motion`; visible focus outlines | Designer |

---

## Validation expectations

Run from the repository root. Everything is read-only apart from a temporary local server.

**JSON validity**

1. `python3 -m json.tool app/project-data.json > /dev/null` must exit 0.
2. `python3 -m json.tool .vscode/launch.json > /dev/null` must exit 0. This shows `launch.json` is strict JSON with no comments or trailing commas.

**Data content**

3. A Python one-liner confirms that `projects` is a non-empty list and that every item has non-empty `name`, `owner`, `status`, `recentActivity` and `priority`. Statuses cover all four values and priorities cover all three.

**Launch configuration**

4. `launch.json` contains:
   - `"name": "Run Project Pulse Dashboard"`
   - `"cwd": "${workspaceFolder}/app"`
   - `"command": "python3 -m http.server 5500"`
   - `serverReadyAction.uriFormat` equal to `http://localhost:%s/index.html`

**Literal strings the workflow checks** (case-insensitive grep)

5. `app/index.html` contains `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity` and `priority`.
6. `app/styles.css` contains `.dashboard`, `.project-card`, `border-radius` and `box-shadow`. It also has selectors for `.status-badge--on-track`, `--at-risk`, `--blocked`, `--completed`, `--unknown`, `.priority-tag--high`, `--medium`, `--low`, and a `@media` rule.
7. **Contract match:** every class in the contract appears in `styles.css`, and every class `styles.css` targets is produced by `index.html` (no orphaned selectors).

**Runtime check**

8. Run `cd app && python3 -m http.server 5500`, then:
   - `curl -sf http://localhost:5500/index.html` returns the HTML
   - `curl -sf http://localhost:5500/project-data.json` returns the JSON
   - stop the server afterwards

**Manual browser check** (the learner, via Run and Debug → **Run Project Pulse Dashboard**)

9. The browser opens `/index.html`, not a directory listing. It shows the "Project Pulse" title, the summary counts and 6 cards. Each card shows status, owner, recent activity and priority, and the long name wraps cleanly.

**Responsive check**

10. In DevTools at 360px, 768px and 1280px widths: 1 column, 2–3 columns, and 3–4 columns respectively. No horizontal scrollbar at any width.

**Accessibility**

11. Check that:
    - there is a single `h1`, and the cards use `h2`
    - `main` is a landmark
    - the message region has `role="status"` / `aria-live`
    - badges contain text
    - Tab shows a visible focus outline
    - text contrast is at least 4.5:1 (DevTools contrast checker or Lighthouse accessibility score ≥ 90)

**Error states** (manual)

12. Opening `app/index.html` via `file://` shows the friendly error. Temporarily testing with `{"projects": []}` (then reverting) shows the empty state.

**Note:** `scripts/validate-exercise.sh` checks the exercise template itself, and it will deliberately fail with "Learner answer files are tracked" once the learner commits these files. Don't use it as the gate for this work.

---

## Open questions

1. **Launch debug type.** The plan uses `node-terminal` because it's built into VS Code and needs no Python extension. If the learner's environment has trouble with it, the alternative is `"type": "debugpy"` with `"module": "http.server"` and `"args": ["5500"]`, but that needs the Python extension, which isn't in `devcontainer.json`. Should we confirm `node-terminal` works in the Codespace during Phase 3?
2. **`file://` fallback.** Should `index.html` include a copy of the data inside the page so it works when double-clicked? The plan says **no**, so the JSON stays the single source of truth, and shows a clear error instead. The Orchestrator should confirm.
3. **Optional `summary` field.** The brief asks for a contributor-friendly summary, but the workflow only requires five fields. The plan treats `summary` as optional but includes it in the sample data. Should it be required?
4. **Status and priority vocabulary.** Is the fixed set ("On Track/At Risk/Blocked/Completed", "High/Medium/Low") acceptable, or does Mona's team use different labels?
5. **Dark mode.** Should the Designer include `prefers-color-scheme: dark`, or keep to a single light theme for this exercise?
6. **Sorting.** Should cards be sorted (for example by priority, then status) or kept in file order? The plan keeps **file order** so the result is predictable, unless the Orchestrator decides otherwise.

---

*Git operations (staging, committing, pushing) are controlled by the learner through Copilot CLI prompts. No agent runs them.*