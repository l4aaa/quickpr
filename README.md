# QuickPR

Reviews every pull request on a local LLM and learns from developer feedback. A GitHub App plus a web dashboard: each opened or updated PR is reviewed by a small local model through Ollama, which posts line-level comments and improves from the feedback developers give on them.

See [docs/spec.md](docs/spec.md) for the specification.

## Stack

ASP.NET Core 11 API, .NET 11 workers, React 19 + Vite, PostgreSQL 19 with pgvector, Redis 8.10, RabbitMQ 4.4, Ollama.

## Prerequisites

- .NET SDK 11
- Node 26 and npm
- Docker Desktop (WSL2 backend; an NVIDIA GPU is used by Ollama and it falls back to CPU)

## Getting started

```sh
cp .env.example .env
docker compose up --build
```

The dashboard is served on http://localhost:8080 and the RabbitMQ management UI on http://localhost:15672.

Run the pieces without Docker:

```sh
dotnet test
cd web && npm ci && npm run dev
```

## Layout

| Path | Contents |
|---|---|
| `src/Api` | Webhook intake, REST API, SignalR |
| `src/Worker` | Review job consumer |
| `web` | React dashboard |
| `tests` | .NET tests |
| `docs` | Specification |
