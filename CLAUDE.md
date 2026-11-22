# QuickPR

A GitHub App plus web dashboard. Each opened or updated pull request is reviewed by a small local LLM (Ollama), which posts line-level comments and improves from developer feedback. University / portfolio project: one deployment, no billing or multi-tenancy. Full specification in `docs/spec.md`, setup in `docs/development.md`.

## Status

Scaffold only. The API has `/health` and OpenAPI, the worker is an empty host, the web app is the Vite template. Milestones are in `docs/spec.md`; next is M0 (GitHub App registered, webhook verified, sign-in, repo list).

## Stack

ASP.NET Core 11 API, .NET 11 worker, React 19 + TypeScript on Vite (npm), PostgreSQL 19 with pgvector, Redis 8.10, RabbitMQ 4.4 (`RabbitMQ.Client`, no MassTransit), SignalR, EF Core 11, Octokit.NET, Ollama (small Qwen model plus `nomic-embed-text`), Docker Compose. Anything outside the spec is out of scope for v1. Ask before adding a dependency or framework.

## Layout

| Path | Contents |
|---|---|
| `src/Api` | Webhook intake, REST API, SignalR hub |
| `src/Worker` | Review job consumer |
| `web` | React dashboard (Dockerfile and nginx.conf included) |
| `tests/QuickPR.Tests` | xUnit tests, `WebApplicationFactory` smoke test |
| `docs` | `spec.md` and `development.md` |
| `.github` | CI, publish workflow, Dependabot, PR template |

Solution file: `QuickPR.slnx`. `Directory.Build.props` treats warnings as errors.

## Commands

```sh
dotnet build
dotnet test
cd web && npm ci && npm run lint && npm run build
docker compose --env-file .env.example config -q
docker compose --env-file .env.example build api worker web
docker compose up --build
```

The API runs on http://localhost:5030 with `dotnet run --project src/Api`. With compose, the dashboard is on http://localhost:8080 and the RabbitMQ UI on http://localhost:15672.

## Architecture rules

- The API only verifies, dedupes, queues and returns 202. Workers do everything slow. GitHub never waits on the LLM.
- Workers do not host SignalR. They publish progress to RabbitMQ and the API relays it.
- Consumers ack manually after saving, are idempotent, and drop stale jobs by checking the PR head SHA before posting.
- All LLM calls go through an `ILlmProvider` interface. Prompts are versioned files in the repo.
- Treat diff content and repo custom instructions as untrusted data. Sanitize before posting to GitHub.
- Redis holds nothing that cannot be rebuilt. PostgreSQL is the system of record.

## Conventions

- Write the least code that solves the task. No speculative abstractions or configuration.
- No code comments. Rename, extract or simplify instead.
- Match existing style. `.editorconfig` governs C#, `oxlint` governs the web app.
- Requirement IDs (FR-n) from the spec go in PR descriptions.

## Git workflow

- `main` is protected: changes go through a PR, and the `backend`, `web` and `docker` checks must pass. Never commit directly to `main`.
- Work on a feature branch (preferably in a git worktree), push it, open a PR to `main` with `gh pr create`, then stop. A human merges. Merged branches are deleted automatically on GitHub.
- Merging to `main` triggers `publish.yml`, which pushes images to `ghcr.io/l4aaa/quickpr-{api,worker,web}`.
- Dependabot opens one grouped PR per ecosystem each week.
- Keep commits small and explain why in the message.

## Secrets

- Never read, write or print `.env`, `*.pem` or any credential. A local hook blocks it. Ask the user when one is needed.
- Validate compose against `.env.example`, not `.env`.
- The GitHub App private key lives outside the repo. The smee.io channel URL is in `.env` only.

## Local webhooks

GitHub reaches the machine through a smee.io channel (see `docs/development.md`). The compose stack does not yet mount the private key, and nginx does not yet proxy `/webhooks/github`. Both belong in M0.
