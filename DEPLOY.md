# Penpot Deployment Runbook — penpot.increstive.co.th

Self-hosted Penpot **2.16** on a DigitalOcean droplet (sgp1) via **Coolify** (Traefik proxy), behind **Cloudflare**, with **Postmark** SMTP and **DigitalOcean Spaces** (S3) object storage.

Files: [`docker-compose.yaml`](./docker-compose.yaml) · [`.env.example`](./.env.example)

---

## What this deployment does

| Requirement | How it's implemented |
|---|---|
| 1. Email+password, scoped to `@increstive.co.th` | Registration left ON + `PENPOT_REGISTRATION_DOMAIN_WHITELIST=increstive.co.th` + `enable-email-whitelist` flag. Open self-service signup, domain-gated (exact match — no subdomains). |
| 2. Email verification required | Verification is ON by default; we **omit** `disable-email-verification`. |
| 3. Email via Postmark SMTP | `enable-smtp` + `smtp.postmarkapp.com:587` STARTTLS; Server Token in **both** username & password; From `Penpot <noreply@increstive.co.th>`. |
| 4. Accessed at `https://penpot.increstive.co.th` | Coolify/Traefik terminates TLS; Cloudflare in front (Full strict). `PENPOT_PUBLIC_URI=https://penpot.increstive.co.th`; secure cookies ON. |
| 5. Files in DO Spaces (S3) | `PENPOT_OBJECTS_STORAGE_BACKEND=s3`, region `sgp1`, endpoint `https://sgp1.digitaloceanspaces.com`, bucket `inc-penpot-contents`. |
| 6. Disable onboarding + MCP server | `disable-onboarding` + `enable-mcp` (with the `penpot-mcp` container) + `enable-access-tokens` (required so MCP clients can authenticate). |

Extras added: `enable-prepl-server` (admin CLI), telemetry OFF, Valkey memory-capped, a stable generated `PENPOT_SECRET_KEY`.

**Final flag string:**
```
enable-smtp enable-email-whitelist disable-onboarding enable-mcp enable-access-tokens enable-prepl-server
```
Deliberately absent: `disable-email-verification`, `disable-secure-session-cookies`, `disable-registration`.

---

## Prerequisites (do these BEFORE deploying)

1. **Droplet**: recommend `s-4vcpu-8gb` in **sgp1** (exporter runs headless Chromium; <4 GB risks OOM on export). You have ~4 users, so 8 GB is comfortable.
2. **DO Space** `inc-penpot-contents` created in **sgp1**, set **Private** (Penpot serves via presigned URLs).
   - Add a **CORS rule**: Origin `https://penpot.increstive.co.th`, methods `GET, PUT, POST, DELETE, HEAD`, allowed headers `*`.
   - Have a Spaces **access key + secret** ready.
3. **Postmark**: domain `increstive.co.th` verified (DKIM + Return-Path) — done. Have the **Server API Token** ready.
4. **Cloudflare** DNS for `penpot.increstive.co.th`:
   - Create the `A` record pointing at the droplet, initially **DNS only (grey cloud)** so Coolify's Let's Encrypt HTTP-01 challenge succeeds.
   - SSL/TLS mode = **Full (strict)**.

---

## Deploy steps (Coolify)

1. **New Resource → Docker Compose** in Coolify; paste `docker-compose.yaml` (or point it at this repo).
2. **Environment Variables**: add the values from `.env.example` — `PENPOT_VERSION`, `PENPOT_SECRET_KEY`, `POSTGRES_PASSWORD` (both pre-generated), `POSTMARK_SERVER_TOKEN`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.
3. **Domain**: on the **`penpot-frontend`** service, set the domain to `https://penpot.increstive.co.th:8080` (the `:8080` is the container port; Coolify serves it on 443). Leave the other services with no domain.
4. **Deploy.** Watch logs until `penpot-backend` reports it has applied DB migrations and is listening.
5. Once the cert is issued (HTTPS works with grey cloud), **flip the Cloudflare record to Proxied (orange)**. Confirm the site still loads and you can log in (this proves Full-strict + secure cookies are consistent).

---

## Post-deploy verification checklist

Work through these in order — each catches a specific, common failure:

- [ ] `https://penpot.increstive.co.th` loads over HTTPS with a valid cert (no warning).
- [ ] **Registration gating**: signing up with a non-`increstive.co.th` email is rejected; an `@increstive.co.th` email is accepted.
- [ ] **Email delivery**: the verification email actually arrives (check Postmark → Activity). If it doesn't:
  - Postmark "From" must be your verified domain — confirm no 422 in Postmark logs.
  - Confirm the token is in **both** SMTP username and password.
- [ ] **Verification enforced**: you cannot log in until the link is clicked.
- [ ] **Asset upload**: create a project, drop in an image. Then verify the object appears in the `inc-penpot-contents` Space.
  - If uploads 500: check backend logs for an **AWS-SDK checksum** error (DO Spaces compatibility) or a **region-validation** error → fallback `PENPOT_OBJECTS_STORAGE_S3_REGION=us-east-1` (keep the sgp1 endpoint).
- [ ] **No onboarding modal** appears for new users.
- [ ] **MCP**: a user can mint a token under *Account → Access tokens*, and an MCP client can connect at the public URL.
- [ ] **Realtime**: open the same file in two tabs — cursors/changes sync (WebSockets working through Cloudflare).
- [ ] **Large upload reality check**: files >100 MB will fail at Cloudflare (Free plan body cap), not Penpot. Plan around it or raise it on Cloudflare.

---

## Operations

- **Backups**: whole-droplet daily snapshot covers the Postgres volume. RPO ≈ 24h; restore is whole-droplet only. If you later want point-in-time / DB-only restore, add a `pg_dump` cron pushing to a **separate** Spaces bucket.
- **Admin CLI**: `enable-prepl-server` lets you run management tasks (e.g. manually verify/promote a user) — `docker exec -ti <backend-container> /opt/penpot/backend/manage.sh` (see Penpot docs → CLI).
- **Upgrades**: bump `PENPOT_VERSION` deliberately (take a snapshot first), redeploy. Never use `latest` — a redeploy could cross a major with DB migrations. All five images (frontend/backend/exporter/mcp + the version) move together.
- **Secret key**: never rotate `PENPOT_SECRET_KEY` casually — it invalidates all sessions and pending verification links.

---

## Decisions record (from the grilling session)

- **TLS**: terminated by Coolify/Traefik (Let's Encrypt), Cloudflare Full-strict in front. No proxy/TLS container inside the compose.
- **Version**: pinned to `2.16` minor (patch-on-redeploy OK; no surprise majors).
- **Registration**: Option A — open self-registration, domain-scoped (not invite-only).
- **Email**: verification required; Postmark domain-verified; single Server Token as user+pass.
- **Storage**: DO Spaces `inc-penpot-contents` @ sgp1, private + presigned URLs + CORS.
- **Database**: in-compose Postgres; durability via daily droplet snapshot (accepted RPO ≈ 24h).
- **MCP**: enabled with `enable-access-tokens` so it's actually usable.
- **Sizing**: 4 users; 4 vCPU / 8 GB droplet recommended.
