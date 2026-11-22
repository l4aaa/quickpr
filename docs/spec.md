# QuickPR specification

University / portfolio project: single deployment, no billing or multi-tenant isolation.

## Goals

- Post a first review on a typical PR (under 300 changed lines) within 5 minutes of the webhook, on a small local model.
- Anchor each comment to a real line in the diff, with a category, a severity and a suggested fix.
- Let teams tune noise per repository (strictness, ignored paths, custom instructions).
- Show the learning loop: retrieved feedback examples appear in later prompts and the share of "not helpful" comments falls as feedback grows.

## Architecture

GitHub webhook → ASP.NET Core 11 API (verify, dedupe, SignalR) → RabbitMQ 4.4 (jobs, retries, DLQ) → .NET 11 workers (chunk, review, post) → Ollama (small local model). PostgreSQL 19 + pgvector is the system of record, Redis 8.10 holds rebuildable state (token cache, rate limits, locks), and the React 19 dashboard updates live over SignalR. The API only accepts and queues work and returns 202; workers do everything slow.

## Review pipeline

1. **Intake (API):** verify signature, record delivery ID, upsert PR, take the Redis dedupe lock, create a review row, publish to `review.requested`.
2. **Fetch (worker):** drop the job if the PR is closed or merged, fetch diff, file list and settings, apply ignore rules and size limits.
3. **Analyze (worker):** chunk the diff (~2k tokens), retrieve feedback examples, call the LLM per chunk, validate the JSON.
4. **Filter (worker):** map findings to diff positions, drop invalid, duplicate or low-value ones, rank and cap.
5. **Persist:** save findings and publish progress events.
6. **Publish:** confirm the head SHA is current, post one GitHub review with inline comments (retry comments alone on 422), complete the check run.

## Functional requirements

P0 is MVP, P1 is v1.

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | Sign in with GitHub; session via secure HTTP-only cookie | P0 |
| FR-2 | Install the GitHub App and list repos the user can admin | P0 |
| FR-3 | Receive and verify webhooks (HMAC-SHA256) for `pull_request`, `installation`, `installation_repositories` | P0 |
| FR-4 | One review job per PR head SHA; skip duplicates and closed or merged PRs | P0 |
| FR-6 | Post a single GitHub review: summary plus inline comments on exact diff lines | P0 |
| FR-7 | Findings have category, severity, message and optional `suggestion` block | P0 |
| FR-8 | Don't repost a finding already posted on an unchanged line | P0 |
| FR-9 | Per-repo settings: strictness, ignored globs, custom instructions | P0 |
| FR-11 | Live progress in the dashboard | P0 |
| FR-12 | Dashboard feedback: helpful / not helpful with optional reason | P1 |
| FR-13 | Use similar past feedback as examples in new reviews (pgvector) | P1 |
| FR-14 | Dashboard statistics per repo | P1 |
| FR-16 | Check run per review: in progress, then neutral | P1 |
| FR-17 | Any language: no language-specific rules | P0 |
| FR-18 | Skip drafts, bot PRs and oversized PRs with an explanatory comment | P0 |

## Milestones

- **M0 Skeleton:** compose with all services, GitHub App registered, webhook verified and logged, GitHub sign-in, repo list. Done when opening a PR creates a review row and a queued job.
- **M1 MVP review:** all P0 requirements, per-file chunking, inline comments with the 422 fallback, settings page, live progress, smoke-test set of 3–5 PRs.
- **M2 Quality and learning:** feedback buttons, shared pgvector retrieval, neutral check runs, GitHub rate limiting.
- **M3 Insights and polish:** statistics page, demo data.

## Decisions

- Local LLM only, a small model on Ollama, swappable through an `ILlmProvider` interface. Review quality is not a goal.
- Feedback learning is shared across all connected repos.
- Limits suit the local model: 20 files, 800 changed lines, ~2k-token chunks.
- Latest stable versions of everything, on purpose.
- RabbitMQ.Client 7.x directly, with retries and dead-lettering by hand (MassTransit v9 is commercial).
