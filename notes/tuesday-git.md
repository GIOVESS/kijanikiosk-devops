# Git Collaboration Practices — KijaniKiosk

## Working Directory vs Staging vs History

The working directory holds your current edits — unstaged changes that git
tracks but has not recorded. The staging area (index) is a snapshot you
explicitly build with `git add` before committing. History is the immutable
chain of committed snapshots. The separation means you can craft precise,
logical commits even when your working directory has unrelated changes in
progress.

## Branching Rules

- `main` — production-ready state only. No direct commits.
- `develop` — integration branch. Feature branches merge here via PR.
- `feature/*` — one concern per branch. Branch from develop, PR back to develop.
- Hotfixes branch from main, merge to both main and develop.

## Pull Request Expectations

Every PR must have: a description explaining what changed and why, at least
one reviewer, and passing CI checks before merge. The PR description is
permanent record — write it for the engineer who inherits this codebase
in 18 months.
