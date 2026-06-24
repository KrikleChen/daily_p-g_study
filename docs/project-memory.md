# Project Memory

## Purpose

This repository records the user's daily PostgreSQL and Greenplum learning. The long-term goal is not only to use PostgreSQL, but to fully understand PostgreSQL internals, PostgreSQL-derived products such as Greenplum / GP, and eventually write PostgreSQL extensions and read or modify database kernel code.

The repository should preserve enough context that future Codex sessions can continue naturally.

## Conversation Summary as of 2026-06-24

### Repository Setup

- Created local repository at `/Users/ymatrix/pg-gp-study-notes`.
- Remote repository: `https://github.com/KrikleChen/daily_p-g_study.git`.
- Main branch: `main`.
- Initial files:
  - `README.md`
  - `.gitignore`
  - `notes/README.md`
  - `notes/2026/2026-06-24.md`

### GitHub Push and Credentials

- Initial push to GitHub succeeded after resolving HTTPS credential issues.
- A repository-local `.git/credentials` file was created earlier for GitHub HTTPS authentication.
- Do not expose or copy tokens into committed files.
- Future Codex sessions should avoid printing tokens and should not change credentials unless the user asks.

### Learning Roadmap

Created the long-term roadmap:

- `docs/pg-gp-kernel-learning-roadmap.md`

The roadmap covers:

- PostgreSQL SQL and DBA basics.
- PostgreSQL source code and kernel flow.
- PostgreSQL extension development.
- Greenplum / GP and MPP database architecture.
- PostgreSQL vs Greenplum comparison.
- Long-term monthly learning phases.

### Skill Installation

Installed several local Codex skills for future work:

- `pdf`
- `jupyter-notebook`
- `security-best-practices`
- `define-goal`
- `cli-creator`
- `mastering-postgresql`

The third-party `mastering-postgresql` skill is application-side focused: PostgreSQL search, pgvector, JSONB, Python drivers, and indexing experiments. It is not a kernel-focused skill.

### PostgreSQL Environment Cleanup and Setup

Cleaned old PostgreSQL instance data:

- Removed old PG 17 data directory: `/Users/ymatrix/data/postgres`.
- Removed stale Homebrew LaunchAgent for `postgresql@17`.
- Did not uninstall the Homebrew `postgresql@17` package.

Later the user installed PostgreSQL 18.

Current observed local database environment:

- Server: Homebrew `postgresql@18`, running on port `5432`.
- Preferred client: `/usr/local/opt/postgresql@18/bin/psql`.
- Shell `psql` may still resolve to `/usr/local/opt/postgresql@17/bin/psql`.
- Use the explicit PG 18 client for exercises.

### Homebrew Update

The user asked to run `brew update`.

Observed issue:

- Homebrew brew repository remote was using the Tsinghua mirror.
- The mirror queued the request for a long time and eventually failed with a bad pack header.

Action taken:

- Changed `/usr/local/Homebrew` origin to official GitHub: `https://github.com/Homebrew/brew.git`.
- `brew update` completed.
- Homebrew version updated to `6.0.3`.
- Bottle/API mirror remained Aliyun.

### EXPLAIN ANALYZE Exercise

Created the exercise:

- `exercises/explain-analyze-order-revenue.md`

Exercise goal:

- Practice reading `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` for a multi-table order revenue query.

Tables:

- `users`
- `products`
- `orders`
- `order_items`

Data volume:

- `users`: 10000 rows.
- `products`: 1000 rows.
- `orders`: 100000 rows.
- `order_items`: 300000 rows.

Query features:

- 4-table join.
- `WHERE` filters.
- `GROUP BY`.
- `HAVING`.
- `ORDER BY`.
- `LIMIT`.
- `count(DISTINCT ...)`.
- Aggregate revenue calculations.

### Homework Review

The user filled in answers in:

- `exercises/explain-analyze-order-revenue.md`

Created review record:

- `reviews/2026-06-24-explain-analyze-homework-review.md`

Key feedback:

- The user can identify major execution plan nodes.
- The user needs to better distinguish estimated rows from actual rows.
- The user mixed some filter metrics between `orders` and `products`.
- The user should not conclude that an index always makes the whole query faster.
- The current score was around `65/100`.

Important concepts explained:

- `Hash Join` is chosen partly because the joins are equality joins, but also because of cost estimates and data volume.
- `HAVING` filters groups after aggregation.
- `top-N heapsort` is an optimization for `ORDER BY ... LIMIT N`, but it may not appear when the final result set is already small.
- `shared hit` means shared buffer hits.
- `shared read` means pages read into shared buffers, not necessarily direct physical disk reads.

### Mandatory Homework Review Process

The user explicitly requested:

> 以后每次教你批改作业，你都需要记录下来批改记录

Created:

- `docs/homework-review-process.md`

Rule:

- Every homework review must create a review file under `reviews/`.
- The review must include verification environment, method, correct reference, correct parts, corrections, answers to questions, mastery evaluation, and next-step advice.

## Current Repository State Notes

As of the latest conversation:

- The user's exercise file may have uncommitted in-progress answer edits.
- Do not overwrite or commit those answer edits unless the user asks.
- Always check `git status --short --branch` before editing.

## How Future Codex Sessions Should Continue

At the start of a session in this repo:

1. Read `AGENTS.md`.
2. Read `README.md`.
3. Read `docs/project-memory.md`.
4. Check `git status --short --branch`.
5. If the user asks for homework review, follow `docs/homework-review-process.md`.

When helping the user learn:

- Prefer Chinese explanations.
- Use local PostgreSQL experiments when useful.
- Save meaningful learning artifacts to the repository.
- Keep review records separate from the user's active homework answers.

