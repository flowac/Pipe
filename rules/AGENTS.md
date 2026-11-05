# Global Multi-Agent Coding Pipeline Rule

- Whenever the user gives a **non-trivial coding task** (implementing a feature, modifying multiple files, architectural refactoring, or debugging a complex issue) OR invokes `/pi`, you **MUST** immediately read and execute the `pi` skill (`C:\Users\win98\.gemini\config\skills\pi\SKILL.md`) to run the 6-stage multi-agent coding pipeline (`1: ingest -> 2: plan -> 3: red-team -> 2: plan rework -> 4: author <-> 5: verify -> 6: review`).
- Only bypass the `pi` pipeline for pure Q&A/explanations, configuration questions, or trivial single-line/typo edits.
