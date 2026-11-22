# Development setup

## Machine

- .NET SDK 11 (pinned in `global.json`), Node 26 with npm, Docker Desktop with the WSL2 backend.
- NVIDIA GPU: only the Windows driver is needed. Ollama runs on CPU if there is no GPU.
- Cap WSL memory so Postgres, Redis, RabbitMQ, Ollama and the .NET services fit. Create `%USERPROFILE%\.wslconfig`, then run `wsl --shutdown`:

```ini
[wsl2]
memory=10GB
```

- Keeping the repo in the WSL filesystem (not `/mnt/c`) is faster. Working from Windows paths also works.

## Environment variables

Copy `.env.example` to `.env`. `.env` is gitignored and must never be committed.

| Variable | Meaning |
|---|---|
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Postgres credentials, local only |
| `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS` | RabbitMQ credentials, local only |
| `OLLAMA_REVIEW_MODEL` | Review model tag. `qwen3.5:4b` is a placeholder, confirm it in M0 |
| `OLLAMA_EMBED_MODEL` | Embedding model, `nomic-embed-text` (768 dimensions) |
| `GITHUB_APP_ID` | App ID from the GitHub App settings page |
| `GITHUB_APP_PRIVATE_KEY_PATH` | Path to the downloaded `.pem`, kept outside the repo |
| `GITHUB_WEBHOOK_SECRET` | Secret entered in the app's webhook settings |
| `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | OAuth credentials of the app, used for sign-in |
| `SMEE_URL` | Your smee.io channel URL, see below |

## Registering the GitHub App

GitHub, Settings, Developer settings, GitHub Apps, New GitHub App.

- **Name:** globally unique, for example `quickpr-<your-login>`.
- **Homepage URL:** the repository URL.
- **Callback URL:** `http://localhost:8080/api/auth/callback`.
- **Request user authorization (OAuth) during installation:** checked.
- **Webhook:** active, URL is the smee channel, secret is a random value you generate (`GITHUB_WEBHOOK_SECRET`). It can be left inactive until webhook intake is built.
- **Repository permissions:** Pull requests read and write, Contents read, Checks write. Metadata read is automatic.
- **Events:** Pull request. `installation` and `installation_repositories` are delivered automatically.
- **Where can this be installed:** only on this account.

Afterwards: note the App ID, the Client ID, generate a client secret, generate a private key (downloads a `.pem`), then install the app on a test repository.

## Local webhooks with smee.io

GitHub can only reach a public URL, so smee.io forwards deliveries to localhost.

1. Open https://smee.io and start a new channel. The URL is permanent. Put it in `.env` as `SMEE_URL`. Anyone with the URL can read its payloads, so use a test repository and do not commit it.
2. Use the same URL as the GitHub App's webhook URL.
3. While testing, run the forwarder. When the API runs with `dotnet run`, it listens on port 5030:

```sh
npx smee-client --url $SMEE_URL --target http://localhost:5030/webhooks/github
```

Deliveries are not queued while the forwarder is stopped. The smee channel page shows recent deliveries and can replay them.

## Not wired yet

- The compose stack does not mount the private key into the `api` and `worker` containers.
- nginx proxies only `/api` and `/hubs`, so `/webhooks/github` is not reachable through port 8080 yet.
- The API and worker contain no features beyond `/health`.

## CI and images

- `ci.yml` runs on every PR and push to `main`: backend build and test, web lint and build, compose validation and image builds. The `backend`, `web` and `docker` checks must pass before merging.
- `publish.yml` runs on pushes to `main` and `v*` tags and pushes `ghcr.io/l4aaa/quickpr-{api,worker,web}`, tagged by commit SHA, semver and `latest`.
