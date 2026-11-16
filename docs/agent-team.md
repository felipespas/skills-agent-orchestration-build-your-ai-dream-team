# Agent team

I will use GitHub Copilot CLI in a Codespace to orchestrate this custom agent team while building Mona's Project Pulse dashboard.

| Agent | Target model | Responsibility | Agent definition |
| --- | --- | --- | --- |
| Orchestrator | Claude Opus 5.5 (copilot) | Coordinates the specialists without implementing code, divides the plan into phases with explicit file ownership, manages dependencies, and verifies the integrated result. | [`.github/agents/orchestrator.agent.md`](../.github/agents/orchestrator.agent.md) |
| Planner | Claude Opus 5.5 (copilot) | Researches the repository and requirements without writing code; produces ordered implementation steps, file assignments, dependencies, edge cases, and validation expectations. | [`.github/agents/planner.agent.md`](../.github/agents/planner.agent.md) |
| Coder | GPT-5.5 (copilot) | Implements dashboard logic and assigned runnable-app support, follows repository patterns, makes behavior deterministic and testable, and validates changes. | [`.github/agents/coder.agent.md`](../.github/agents/coder.agent.md) |
| Designer | Gemini 3.7 Flash (copilot) | Creates a polished, accessible, responsive dashboard with clear information hierarchy, project cards, status badges, priority treatment, and consistent visual styling. | [`.github/agents/designer.agent.md`](../.github/agents/designer.agent.md) |

The Orchestrator starts with the Planner's plan, then delegates scoped work to the Coder and Designer. Independent tasks may run in parallel only when files do not overlap and there are no data dependencies; dependent or overlapping work runs sequentially. The learner controls staging, commits, and pushes through Copilot CLI prompts.
