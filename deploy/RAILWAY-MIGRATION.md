# n8n → Railway Migration Runbook

Move this **self-hosted n8n** (local Podman/`docker-compose.yml`, exposed via a fragile unnamed
Cloudflare tunnel) to **Railway**. This is a **migration, not a fresh install** — workflows *and*
encrypted credentials must survive.

Files in this folder:
- [`Dockerfile`](./Dockerfile) — the n8n service image (point Railway's Root Directory here)
- [`railway.env.example`](./railway.env.example) — the env vars to set on the n8n service

---

## How this maps from `docker-compose.yml`

Railway does **not** run `docker-compose`. It deploys one service per image/Dockerfile. The
three compose services become:

| Compose service | On Railway |
|---|---|
| `n8n` | a service built from [`Dockerfile`](./Dockerfile) |
| `postgres` | **moved off Railway** → a dedicated **Supabase Cloud** Postgres project (free tier). Keeps the bill to one Railway service (Trim B). |
| `n8n-exporter` | **dropped** — the Supabase DB is durable; a second always-on n8n just to loop `export:workflow` is wasted cost. (Want git-versioned JSON still? Run `railway run n8n export:workflow --all --output=...` on demand, or add an in-n8n scheduled workflow that commits to GitHub.) |

---

## Steps

### 0. Know your real local values
The commands below use placeholders. Your actual Postgres role/db are whatever `.env` sets for
`POSTGRES_USER` / `POSTGRES_DB` (n8n's own `N8N_ENCRYPTION_KEY` is also in there) — `.env` is only
read by `podman-compose`, it is **not** loaded into your shell, so `$POSTGRES_USER` is empty if you
type it directly in PowerShell. Check the file once, then use the literal values below:
```powershell
Select-String '^POSTGRES_USER|^POSTGRES_DB|^N8N_ENCRYPTION_KEY' .env
```

### 1. Pin the n8n version
Capture the running version and set it in [`Dockerfile`](./Dockerfile) (`FROM n8nio/n8n:X.Y.Z`):
```powershell
podman exec aa_n8n n8n --version
```
Pin forward-or-equal only — never deploy an older tag than your data was written with.

### 2. Create the database (Supabase) and the n8n service (Railway)
1. **Database — Supabase Cloud (free).** Create a **dedicated** Supabase project for n8n (so the
   restore in step 4 is a clean `public`→`public` copy with no coupling to any other app, and its
   own 500 MB). From **Project Settings → Database → Connection string → Session pooler**, note:
   host `aws-0-<region>.pooler.supabase.com`, port `5432`, user `postgres.<ref>`, and the DB
   password. (Session pooler = IPv4 + persistent connections; the Transaction pooler on 6543 breaks
   n8n's prepared statements, and the direct host is IPv6-only.)
2. **n8n service — Railway.** **New → GitHub Repo** (this repo) or **Empty Service → Deploy from
   Dockerfile**. In **Settings → Build**, set **Root Directory = `deploy`** so it builds the
   Dockerfile here.
3. **Settings → Networking → Generate Domain.** Confirm the target port is **5678**.

### 3. Set the env vars
Copy [`railway.env.example`](./railway.env.example) into the n8n service's
**Variables → Raw Editor** and fill in real values. The non-negotiable one:

- **`N8N_ENCRYPTION_KEY` must equal the old instance's value** (from this repo's `.env`, step 0).
  Mismatch = all stored credentials become unreadable.

Fill `N8N_HOST` / `WEBHOOK_URL` / `N8N_EDITOR_BASE_URL` with the domain from step 2.3.

### 4. Migrate the data (Postgres → Postgres)
Because both old and new are Postgres, dump & restore — this preserves workflows, executions,
**and** encrypted credentials (which is why step 3's key must match). Substitute the real
`POSTGRES_USER` / `POSTGRES_DB` from step 0 (commonly `n8n_user` / `n8n` per `example.env`):

```powershell
# from this repo's root — dump the current n8n DB (literal values, not $POSTGRES_USER — see step 0)
podman exec aa_postgres pg_dump -U n8n_user -d n8n --no-owner --clean > n8n_backup.sql

# restore into the DEDICATED Supabase project (Session-pooler connection string)
psql "postgresql://postgres.<ref>:<password>@aws-0-<region>.pooler.supabase.com:5432/postgres" -f n8n_backup.sql
```
> Using a dedicated Supabase project keeps this a trivial `public`→`public` copy. If you instead
> share another project's Supabase database, first `CREATE SCHEMA n8n;` there, set
> `DB_POSTGRESDB_SCHEMA=n8n` in the env, and remap the dump into that schema (e.g.
> `(Get-Content n8n_backup.sql) -replace 'public\.', 'n8n.' | Set-Content n8n_backup_remapped.sql`)
> — which is exactly the coupling/extra step the dedicated project avoids.

### 5. Deploy & verify
Deploy the n8n service. It should connect to the restored DB and show your workflows + credentials.
- Log in (owner account came with the DB migration).
- Spot-check a credential opens without a decryption error → confirms the key matched.

### 6. Re-point the webhooks (they don't update themselves)
- **Telegram** — re-register the bot webhook to the new URL:
  ```powershell
  curl.exe -s "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-n8n-subdomain>.up.railway.app/webhook/<your-telegram-webhook-path>"
  ```
- **Facebook / any other** inbound webhooks pointing at the old Cloudflare tunnel → update to the
  Railway domain.
- Activate the workflows.

### 7. (Optional) persistence insurance
Attach a small **Railway Volume at `/home/node/.n8n`** on the n8n service for binary-data
durability across redeploys. Not strictly required (DB on Postgres + key in env), but cheap.

---

## Cost note (Trim B)
Only **one** billable Railway resource is added — the n8n service itself; its Postgres lives on the
**Supabase Cloud free tier**, off the Railway bill. n8n **cannot sleep** (it must catch Telegram
webhooks and run the slot-picker cron), so it's a continuous RAM charge. Budget **~$5–7/month
total**. The dedicated Supabase project gets its own 500 MB, the 14-day `EXECUTIONS_DATA_PRUNE`
keeps execution data bounded, and n8n's constant traffic keeps the free project from auto-pausing.

## Pre-existing workflow TODOs (not migration-blocking, good time to fix)
- Double-tap guard on the serialized slot-picker sub-flow (no claim-write step at the top to
  prevent duplicate publishing if triggered twice in quick succession).
- Verify no hardcoded personal Telegram `chatId` remains before using this for group-based approval.
- Templated.io paid upgrade — still pending, required for production rendering volume.
