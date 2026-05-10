# Requirements Docs

This folder holds Dotty's canonical requirements documents. The **authoritative versions** live in this folder; the root `README.md` and `AGENTS.md` reference them.

| Document | Role |
|----------|------|
| `PRD-Dotty.md` | **Product Requirements Document.** What to build. Architecture decisions, component inventories, blocking questions, reference specs. |
| `TRD-Dotty.md` | **Technical Requirements Document.** Implementation spec derived from the PRD. 8 vertical slices with entry/exit criteria. |

For handoff context, locked decisions, and slice status, see the root `AGENTS.md`. For a high-level overview of the project, see the root `README.md`.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | May 2026 | Initial TRD after architecture Q&A |
| 1.1 | May 2026 | System prompt architecture, disambiguation rules, builtin tools, token budget, event mapping, slice entry/exit criteria, project structure |