# Codex Project Memory

## Project Purpose

This repository is the user's daily learning record for PostgreSQL, Greenplum / GP, PostgreSQL-derived database products, and eventually PostgreSQL extension and kernel development.

The user wants this project to act as a persistent learning workspace. Future Codex sessions opened in this repo should recover the project context from this file and the linked docs before making changes.

## Language and Style

- Write learning notes, reviews, and explanations in Chinese unless the user explicitly asks otherwise.
- Keep SQL, command names, function names, and execution plan node names in their original English form.
- Prefer practical explanations based on local experiments, not abstract summaries.
- When teaching PostgreSQL, focus on command, observation, explanation, and conclusion.

## Repository Structure

- `README.md`: top-level entry point.
- `docs/pg-gp-kernel-learning-roadmap.md`: long-term PostgreSQL, Greenplum, kernel, and extension learning roadmap.
- `docs/homework-review-process.md`: mandatory process for homework reviews.
- `docs/project-memory.md`: project memory and conversation summary.
- `notes/`: daily study notes.
- `exercises/`: hands-on exercises.
- `reviews/`: homework review records.

## Must-Read Files at Session Start

When working in this repo, first inspect:

- `README.md`
- `docs/project-memory.md`
- `docs/homework-review-process.md`
- the relevant file under `exercises/`, `notes/`, or `reviews/`

Also run `git status --short --branch` before editing. The user may have in-progress answers that should not be overwritten or accidentally committed.

## Homework Review Rule

The user explicitly requires: every time Codex reviews homework, Codex must record the review in the local repository.

Follow `docs/homework-review-process.md`.

Default behavior for homework review:

- Read the user's current exercise or note.
- If verification needs PostgreSQL, use the local `test` database when appropriate.
- Prefer a temporary schema for destructive or setup SQL.
- Do not delete or overwrite the user's `public` schema tables unless explicitly asked.
- Write a review file under `reviews/`.
- Keep the user's unfinished homework edits separate from review commits unless the user asks to commit them.

## Current PostgreSQL Environment Notes

As of 2026-06-24:

- PostgreSQL server: Homebrew `postgresql@18`, running on local port `5432`.
- Preferred client: `/usr/local/opt/postgresql@18/bin/psql`.
- The shell `psql` in PATH may still point to PostgreSQL 17, so use the explicit PG 18 client when version consistency matters.
- Test database: `test`.
- In `test.public`, the exercise tables currently exist:
  - `users`: 10000 rows.
  - `products`: 1000 rows.
  - `orders`: 100000 rows.
  - `order_items`: 300000 rows.
- Exercise indexes currently present:
  - `idx_orders_status_created_user`
  - `idx_order_items_order_id`
  - `idx_order_items_product_id`
  - `idx_products_category`

## Current Learning State

The user is working on an `EXPLAIN ANALYZE` exercise for a multi-table order revenue query.

Current strengths:

- Can identify `Seq Scan`, `Bitmap Heap Scan`, `Bitmap Index Scan`, `Hash Join`, `GroupAggregate`, `Sort`, and `Limit`.
- Understands that `ANALYZE` collects optimizer statistics.
- Understands that low selectivity can make indexes unattractive.

Current focus areas:

- Distinguish estimated `rows` from actual `rows`.
- Assign each metric to the correct plan node.
- Do not mix `orders` filter rows with `products` filter rows.
- Understand that `HAVING` filters groups after aggregation.
- Compare plans beyond just checking whether an index was used.

Most recent homework review score: around `65/100`.

## Git and Remote Notes

- Remote: `origin` points to `https://github.com/KrikleChen/daily_p-g_study.git`.
- Do not push unless the user asks.
- Avoid committing the user's in-progress answer edits unless explicitly requested.
- Review and process docs can be committed separately.

## Local Tooling Notes

During this project setup, several Codex skills were installed locally, including:

- `pdf`
- `jupyter-notebook`
- `security-best-practices`
- `define-goal`
- `cli-creator`
- `mastering-postgresql`

Use them only when their trigger conditions fit the task.

