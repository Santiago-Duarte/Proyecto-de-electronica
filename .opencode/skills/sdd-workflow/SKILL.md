---
name: sdd-workflow
description: Spec-Driven Development workflow for this repo. Use when creating, planning, reviewing or implementing a feature, or when asked how specs, plans and tasks should be written.
---
# SDD workflow

Order is strict: constitution -> spec -> plan -> tasks -> implementation.

- spec.md says WHAT and WHY. No stack, no file paths. Acceptance criteria must be verifiable.
- plan.md says HOW. It must pass docs/constitution.md; justify any exception.
- tasks.md is a checklist of small tasks, each linked to an acceptance criterion.
- Code only follows approved specs. If reality contradicts a spec, update the spec first.
- Log decisions and lessons in docs/memory/. Write documents in the language set in docs/constitution.md.
