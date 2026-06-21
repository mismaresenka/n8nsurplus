# n8n for Railway — A&A pipeline (migrated from the local Podman/compose stack).
#
# Railway deploys ONE service from this Dockerfile. The Postgres database lives
# OFF Railway on the Supabase Cloud free tier (Trim B — see RAILWAY-MIGRATION.md),
# and the old `n8n-exporter` sidecar is intentionally dropped (the Supabase DB is
# already durable). All configuration and secrets are set as Railway
# environment variables — see ./railway.env.example — NOT baked into this image.
#
# Point Railway's service "Root Directory" at deploy/n8n so it builds this file.
#
# PIN the tag to the version your existing instance runs, for a clean migration:
#   podman exec aa_n8n n8n --version
# Forward upgrades auto-migrate the DB schema; never deploy an OLDER tag than the
# version your data was written with. If you were on `:latest` locally, capture
# the exact version NOW (command above) and pin it here before migrating.
FROM n8nio/n8n:2.25.7

# n8n listens on 5678 by default. EXPOSE lets Railway auto-detect the target
# port for the generated domain (confirm it under Settings → Networking).
EXPOSE 5678
