# bwb E-Commerce Platform — Project Docs

Planning, architecture, and task-tracking docs for an e-commerce/POS/inventory platform built
on SELISE Blocks. **No app code lives here** — see [AGENTS.md](AGENTS.md) for the three
sibling repos that hold the actual apps, and for the full context an AI coding assistant
(Claude Code, Codex, etc.) needs to continue this work on a new machine.

## Start here

1. [AGENTS.md](AGENTS.md) — full context: the hard constraints, where the actual code lives,
   what's safe to do vs. what needs your explicit go-ahead.
2. [ECOMMERCE_TASK_BREAKDOWN.md](ECOMMERCE_TASK_BREAKDOWN.md) — the live tracker. What's done,
   what's next, one checkbox per task.

## ⚠️ Before you switch machines

Commit and push the in-progress work in `ecommerce-back-office` and `ecommerce-consumer`
first — see the warning in [AGENTS.md](AGENTS.md). This repo is docs only; the actual code
changes live in those repos' working trees until committed.

## Everything else in this folder

See the table in [AGENTS.md](AGENTS.md) — architecture reference, platform gap tracking, a
security audit, the live schema export, and two ready-to-import packages
(`COMMERCE_SCHEMAS_DRAFT.*`, `P0_POLICY_FIXES.*`) waiting on your own `blocks` CLI session.
