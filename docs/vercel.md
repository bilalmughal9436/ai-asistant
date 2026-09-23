# Deploying the UI to Vercel

Open Executive has two independently deployed parts:

- the stateless Next.js UI, which can run on Vercel;
- the stateful FastAPI service, which must run as one persistent container with
  a volume mounted at `/data`.

The repository-root `vercel.json` makes a direct Vercel import build the nested
UI in `packages/ui`. Do not deploy the Python service as a Vercel Function: its
SQLite/Chroma state, background scheduler, memory use, and long startup do not
fit that runtime.

## 1. Deploy the API first

Follow [deployment.md](deployment.md) on a container host that supports a
persistent volume. Give the API a public HTTPS address and set:

```text
OE_PUBLIC_DEPLOYMENT=1
BACKEND_SHARED_SECRET=<a-long-random-secret>
BACKEND_ALLOWED_ORIGINS=https://your-project.vercel.app
```

Keep the API at exactly one replica. Verify `https://your-api.example.com/health`
before deploying the UI.

## 2. Import this repository into Vercel

Use the repository root as the Vercel Root Directory. The checked-in
`vercel.json` supplies the nested install, build, and output settings.

Add these variables to Production (and Preview too, if previews should work):

| Variable | Value |
|---|---|
| `BACKEND_BASE_URL` | Public API origin, without a trailing slash |
| `BACKEND_SHARED_SECRET` | Exactly the same secret used by the API |
| `AUTH_SECRET` | A strong random value, for example `openssl rand -base64 32` |
| `AUTH_GOOGLE_ID` | Google OAuth web client ID |
| `AUTH_GOOGLE_SECRET` | Google OAuth client secret |
| `AUTH_URL` | Canonical Vercel UI origin, such as `https://exec.example.com` |
| `AUTH_TRUST_HOST` | `true` |
| `ALLOWED_EMAILS` | Comma-separated Google account emails allowed to sign in |

All of these are server-only variables; none should use a `NEXT_PUBLIC_` prefix.

In the Google OAuth client, add this exact authorized redirect URI:

```text
https://YOUR-UI-DOMAIN/api/auth/callback/google
```

Then deploy. The browser only calls `/api/backend/*` on the Vercel origin; the
Next.js server proxy calls `BACKEND_BASE_URL` and adds the shared secret.

## Troubleshooting

- **Build runs from the wrong folder:** keep Vercel's Root Directory at the
  repository root so `vercel.json` is discovered. Do not also set it to
  `packages/ui` unless you remove/override the root config.
- **OAuth redirects to the wrong host:** set `AUTH_URL` to the exact production
  origin and update Google's callback URI to match it.
- **UI shows backend errors:** confirm the API URL is publicly reachable from
  Vercel and both services have the identical `BACKEND_SHARED_SECRET`.
- **CORS failure:** add the exact Vercel/custom UI origin to
  `BACKEND_ALLOWED_ORIGINS` on the API and restart it.
- **Preview deployment sign-in fails:** Google OAuth needs an exact callback
  URL. Use a stable preview/custom domain or add the preview callback explicitly.

