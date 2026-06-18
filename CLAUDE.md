# cp-workflow-template — Claude Code Guidelines

## Git Workflow

This repo has **no `dev` branch** — `master` is the main branch.

**Never push directly to `master`.** Always:
1. Create a feature branch from `master` (e.g. `fix/short-desc`, `feat/short-desc`)
2. Make changes on the feature branch
3. Open a PR targeting `master`
4. Merge via PR

Direct pushes to `master` are only allowed in emergencies (e.g. a hotfix that unblocks production CI for all downstream repos), and must be explicitly confirmed by the user each time.
