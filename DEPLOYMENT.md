# FlowChart AI — Self-Hosting / Deployment Guide

This guide covers deploying FlowChart AI to **Railway** with a **Supabase**
Postgres database. For the full list of environment variables see
[`.env.example`](./.env.example).

---

## Architecture at a glance

- **App:** Next.js 15 (App Router) — runs as a Node server (`pnpm build` / `pnpm start`).
- **DB:** PostgreSQL via Drizzle ORM. Connects with a direct connection string
  (`DATABASE_URL`), **not** the Supabase client/anon key.
- **Auth:** Better Auth (email/password + optional Google/GitHub OAuth).
- **AI:** OpenRouter API.
- **Email:** Resend (required for email verification, which is enforced).
- **Storage:** S3-compatible (Cloudflare R2 / AWS S3) for image uploads + thumbnails.
- **Payments:** Creem (optional).

---

## 1. Database (Supabase)

A Supabase project has been provisioned for this deployment:

| Field | Value |
|---|---|
| Project name | `flowchartai` |
| Project ref | `mpkbzrzfbfviiferujbc` |
| Region | `us-east-2` |
| API URL | `https://mpkbzrzfbfviiferujbc.supabase.co` |
| Schema | All 9 tables migrated (Drizzle `0000`–`0006`) |

### Build the `DATABASE_URL`

The app uses `postgres-js` with `prepare: false`, which is compatible with the
Supabase connection pooler. Get the password from **Supabase Dashboard →
Project Settings → Database** (or reset it there), then use the pooler string:

```
# Session pooler (recommended for Railway's persistent Node server)
DATABASE_URL="postgresql://postgres.mpkbzrzfbfviiferujbc:[PASSWORD]@aws-0-us-east-2.pooler.supabase.com:5432/postgres"
```

> The exact pooler host/port is shown under **Connect** in the Supabase
> dashboard. The direct connection (`db.mpkbzrzfbfviiferujbc.supabase.co:5432`)
> also works for Railway since it runs a long-lived server.

> ⚠️ **Security — Row Level Security (RLS) is disabled** on all 9 tables.
> The app connects as the `postgres` role over a direct connection, which
> **bypasses RLS**, so enabling RLS will *not* break the app — but it closes off
> the auto-generated public PostgREST/anon API. Recommended hardening (run in the
> Supabase SQL editor):
>
> ```sql
> ALTER TABLE public.account ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.payment ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.session ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.user ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.verification ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.credits_history ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.flowcharts ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.ai_usage ENABLE ROW LEVEL SECURITY;
> ALTER TABLE public.guest_usage ENABLE ROW LEVEL SECURITY;
> ```

### Future schema changes

Migrations live in `src/db/migrations`. To apply new ones against the DB:

```bash
pnpm db:generate   # after editing src/db/schema.ts
pnpm db:migrate    # applies pending migrations using DATABASE_URL
```

---

## 2. Required environment variables (minimum to boot)

| Variable | Notes |
|---|---|
| `NEXT_PUBLIC_BASE_URL` | Public URL of the deployment (e.g. `https://flowchartai.up.railway.app`). No trailing slash. |
| `DATABASE_URL` | From step 1. |
| `AUTH_SECRET` | `openssl rand -base64 32` |
| `OPENROUTER_API_KEY` | Required for the core AI feature. |

**At least one working login path** is also required:
- OAuth: `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` and/or `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET`, **or**
- Email/password: `RESEND_API_KEY` (email verification is enforced, so mail must work).

Image-to-flowchart + uploads additionally need the `STORAGE_*` (S3/R2) vars.

> Verify the model slugs in `src/lib/ai-models.ts`
> (`deepseek/deepseek-v4-flash`, `bytedance-seed/seed-2.0-mini`) resolve on
> OpenRouter before going live, or the AI calls will fail.

---

## 3. Deploy to Railway

Railway runs the full Node server (best fit for this app). `railway.json` is
included.

1. Create a project → **Deploy from GitHub repo** → select this repo and the
   deployment branch.
2. Railway auto-detects Next.js (Nixpacks). `railway.json` pins the build
   (`pnpm build`) and start (`pnpm start`) commands; `.nvmrc` pins Node 22.
3. Add the environment variables from step 2 under **Variables**.
4. Generate a domain under **Settings → Networking**, then set
   `NEXT_PUBLIC_BASE_URL` to that domain and redeploy.
5. Railway provides `PORT` automatically — `next start` honors it.

---

## 4. OAuth callback URLs

Set these in the Google / GitHub OAuth app config (replace the host):

- Google: `<NEXT_PUBLIC_BASE_URL>/api/auth/callback/google`
- GitHub: `<NEXT_PUBLIC_BASE_URL>/api/auth/callback/github`

---

## 5. Build-time dependency on `cdn.flowchartai.org` (important)

The `pnpm build` step runs `content-collections build`, which compiles the MDX
blog/docs content. Fumadocs' `remarkImage` plugin probes the **dimensions of
remote images at build time** and, on failure, **re-throws** — so if any
referenced image URL is unreachable during the build, the entire build fails
with `Cannot process MDX file with esbuild`.

The bundled blog content references images on `https://cdn.flowchartai.org/...`.
That CDN is reachable from normal build environments (including Railway), so the
build works there. Be aware:

- Build environments with **no/blocked outbound internet** will fail this step.
- If that CDN ever changes or the images move, the build breaks even though no
  code changed.

**Hardening options for a long-lived fork** (optional):
- Re-host the blog images on your own storage/CDN and update the MDX references, **or**
- Disable remote image-size probing by passing a custom `remarkImageOptions`
  (e.g. `{ external: false }`) to the Fumadocs MDX transform in
  `content-collections.ts`, **or**
- Remove/trim the bundled marketing blog content you don't need under `content/`.

## 6. Post-deploy checklist

- [ ] `DATABASE_URL` set (password filled in) and app connects.
- [ ] `AUTH_SECRET`, `OPENROUTER_API_KEY` set.
- [ ] At least one login path configured (OAuth or Resend).
- [ ] `NEXT_PUBLIC_BASE_URL` matches the real domain; OAuth callbacks updated.
- [ ] OpenRouter model slugs verified.
- [ ] (Recommended) RLS enabled on Supabase tables.
- [ ] (Optional) `STORAGE_*` for image uploads; Creem for paid plans.
