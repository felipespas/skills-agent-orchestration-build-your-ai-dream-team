# Project Pulse final handoff

Mona's Project Pulse dashboard is a static frontend (no build tools) built with GitHub Copilot CLI in a Codespace. This document is the final handoff for the work.

## Agent team

| Agent | Role in this build |
| --- | --- |
| Orchestrator | Coordinated the work, assigned non-overlapping file scopes, ran the Designer and Coder in parallel, and validated the result. |
| Planner | Researched the repository and produced `docs/project-pulse-plan.md` (shared contract, file assignments, dependencies, parallel work decisions, validation expectations). |
| Designer | Owned visual and accessibility decisions and wrote `app/styles.css`. |
| Coder | Wrote `app/index.html`, `app/project-data.json`, and `.vscode/launch.json`. |

Agent definitions live in `.github/agents/`. Team details are in `docs/agent-team.md`.

## Deliverables

| File | Owner | Purpose |
| --- | --- | --- |
| `app/index.html` | Coder | Page shell (title "Project Pulse"), fetches `project-data.json`, renders one `project-card` per project with status, recentActivity, and priority, and handles loading, empty, and error states. |
| `app/styles.css` | Designer | Polished responsive styling: `.dashboard` and `.project-card` selectors, `border-radius`, `box-shadow`, grid layout, status badges, priority tags, focus styles, reduced motion, dark mode. |
| `app/project-data.json` | Coder | Top-level `projects` key with 6 projects, each with name, owner, status, recentActivity, priority, and summary. |
| `.vscode/launch.json` | Coder | Strict JSON launch file. |

## How to run

1. Open Run and Debug in VS Code.
2. Start the launch configuration named "Run Project Pulse Dashboard" from `.vscode/launch.json`.
3. It runs `python3 -m http.server 5500` with `cwd` set to `${workspaceFolder}/app`, then `serverReadyAction` opens `http://localhost:%s/index.html`, so the dashboard opens rather than a directory listing.
4. Stop the server by ending the debug session or terminal.

Opening `app/index.html` directly through `file://` will not load the data, because the browser blocks `fetch`. Use the launch configuration.

## Dashboard validation results

Checks run by the Orchestrator:

- `app/project-data.json` parses; it has 6 projects covering all four statuses (On Track, At Risk, Blocked, Completed) and all three priorities (High, Medium, Low); every project has name, owner, status, recentActivity, and priority.
- `.vscode/launch.json` parses as strict JSON, contains the name "Run Project Pulse Dashboard", the command `python3 -m http.server 5500`, the `cwd` `${workspaceFolder}/app`, and a `serverReadyAction` with `http://localhost:%s/index.html`.
- `app/index.html` has the exact title `Project Pulse`, references `styles.css` and `project-data.json`, and contains no `innerHTML` and no inline `style=` attributes.
- `app/styles.css` contains the `.dashboard` and `.project-card` selectors, `border-radius`, `box-shadow`, and `@media` rules.
- Serving `app/` on port 5500 returned HTTP 200 for `index.html`, `styles.css`, and `project-data.json`.
- A jsdom render test of `app/index.html` with the real data produced 6 `.project-card` elements, 6 status badges, 6 priority tags, a hidden message region, and header counts of 2 On Track, 2 At Risk, 1 Blocked, 1 Completed.

Not validated:

- Visual appearance in a real browser, including responsive breakpoints, dark mode, and color contrast (the Designer targeted 4.5:1 but it was not measured).
- Running the launch configuration in VS Code, including that `node-terminal` and `serverReadyAction` behave as expected in the Codespace.
- Keyboard navigation and screen reader behavior.

## Known risks

- If port 5500 is already in use, the server will not start and the browser will not open.
- A status outside the four known values is shown as written but is not counted in the header totals.
- The Coder split the "Recent activity" row into `.project-card__activity-label` and `.project-card__activity-text`; the Designer's CSS styles them through generic first-child rules, so check their rendering.

## Next steps for the handoff

1. Run the launch configuration and review the dashboard in a browser at several widths.
2. Run an accessibility check (for example Lighthouse or axe).
3. Replace the sample project data in `app/project-data.json` with real projects.

Git operations (staging, committing, pushing) stay with the learner through Copilot CLI prompts.
