# AGENTS.md

## Project

This repository contains the MyProject application.

Before making significant changes, read:

- docs/PROJECT.md
- docs/ARCHITECTURE.md
- docs/DEVELOPMENT.md

For feature work, read the corresponding directory under:

- specs/

## Development Rules

- Do not change architecture without explicit approval.
- Do not introduce new dependencies unless necessary.
- Keep changes scoped to the current task.
- Follow existing project patterns before introducing new patterns.
- Do not modify unrelated files.

## Planning

For non-trivial features or refactors:

1. inspect the repository first
2. create or update a PLAN.md
3. identify affected files
4. define validation steps
5. wait for approval before implementation

## Verification

Before marking a task complete, run:

- lint
- typecheck
- unit tests
- relevant integration tests
- build

Do not continue to the next task if validation fails.

## Documentation

If implementation changes:

- architecture
- project behavior
- APIs
- important technical decisions

update the relevant documentation.

## Git

Do not commit or push unless explicitly requested.