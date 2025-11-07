# Papercrate Demo

A docker-compose setup for trying Papercrate locally.

## Configuration
- Copy `.env` (or create it) before running Compose; the file is loaded automatically and must at least define `POSTGRES_PASSWORD`, `JWT_SECRET`, `MINIO_ROOT_PASSWORD`, plus the WebAuthn values `WEBAUTHN_ORIGIN`, `WEBAUTHN_RP_ID`, and `WEBAUTHN_RP_NAME`.
- Set the WebAuthn origin/host pair to either `localhost` (only works without TLS) or a real HTTPS site; Papercrate itself does not terminate TLS, so you must front the stack with your own HTTPS ingress (ngrok, Traefik, router TLS, etc.) for any non-localhost hostname. Passkeys are the only authentication option, so secure-context requirements must be satisfied.
- Any other overrides from `docker-compose.yml` (ports, bucket name, etc.) can live in the same `.env` file.
- `PROXY_DOWNLOADS=true` keeps MinIO private by streaming downloads through the backend. Set it to `false` only if you plan to expose MinIO (or point `AWS_ENDPOINT_URL` at another S3 endpoint) and want the browser to use presigned URLs directly.
- WebDAV (currently read-only) is exposed via the `webdav` service on `${WEBDAV_PORT:-3001}`. Generate an access token in the UI (Settings → API Tokens → Create Token, capability `webdav`) and connect any WebDAV client to `http://localhost:3001/` using your Papercrate username plus that token as the password.

## Quick Start
```
docker compose up -d   # first startup pulls images, runs migrations, and may take a minute
docker compose exec backend /usr/local/bin/papercrate-admin create-user demo
docker compose exec backend /usr/local/bin/papercrate-admin create-tenant demo
# replace <TENANT_ID> with the value from the previous command
docker compose exec backend /usr/local/bin/papercrate-admin add-user-to-tenant demo <TENANT_ID>
docker compose exec backend /usr/local/bin/papercrate-admin magic-token demo
```
- `magic-token` prints a one-time login token for the user; on the default setup open `http://localhost:8080/#/account/login/?magic_token=<TOKEN>` (replace the placeholder). The token stays valid for 10 minutes and establishes a browser session cookie that lasts up to 30 days, after which you’ll need a new magic token or a passkey login (configure `WEBAUTHN_ORIGIN` with an HTTPS host you control).
- By default the frontend listens on `${FRONTEND_PORT:-8080}` over HTTP. For passkeys on anything other than localhost, run the frontend behind your own TLS terminator that matches `WEBAUTHN_ORIGIN`.
- Users and tenants are distinct objects: users authenticate and can belong to multiple tenants. The quick start uses the name `demo` for both, but membership is only established once you run `add-user-to-tenant`.

## Admin CLI
- All admin operations run inside the backend container, e.g. `docker compose exec backend /usr/local/bin/papercrate-admin <command> [args]`.
- List available subcommands or arguments with `docker compose exec backend /usr/local/bin/papercrate-admin --help`.
- Common tasks include `create-user`, `create-tenant`, `add-user-to-tenant`, `magic-token`, `users list`, and `tenants list`; each subcommand supports `--help` for detailed flags.

## Data
- Uploaded files and derived blobs live in the `minio_data` volume, so `docker compose down -v` will wipe them.
- Postgres (`postgres_data`) and Quickwit (`quickwit_data`) also persist via local Docker volumes should you want a full reset.
