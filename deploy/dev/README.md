# deploy/dev — disposable Cloud Run deployment (apps/web)

Dev-only deployment of the web dashboard to Google Cloud Run. This is **not**
the qualified WD-024 runtime: `infra/runtime/` stays authored-not-applied and
fail-closed until the platform owner records explicit environment
authorization. Nothing here touches live databases or production secrets.

| Item | Value |
| --- | --- |
| GCP project | `ut-gee-wri-hydro-dev` |
| Region | `us-west1` |
| Artifact Registry | `utah-water` (docker) |
| Cloud Run service | `utah-water-web` |
| Container | `nginxinc/nginx-unprivileged:1.27-alpine` serving the Vite `dist/` |

## Files

- `Dockerfile.web` — multi-stage: node:22-alpine + corepack/pnpm workspace
  install → `pnpm --filter @utah-water/web run build` → nginx serve.
- `nginx.conf` — SPA fallback (`try_files … /index.html`), immutable cache for
  `/assets/`, listens on 8080 (matches `PORT` injected by Cloud Run).
- `cloudbuild.web.yaml` — builds + pushes
  `us-west1-docker.pkg.dev/ut-gee-wri-hydro-dev/utah-water/web:dev-<sha>` and
  moving tag `:dev`.

## Commands (from repo root)

```bash
# 1. Build & push image (tag = git sha; $SHORT_SHA is empty for tarball uploads)
gcloud builds submit --config deploy/dev/cloudbuild.web.yaml \
  --substitutions=_IMAGE_TAG=$(git rev-parse --short HEAD) .

# 2. Deploy (first time creates the service)
gcloud run deploy utah-water-web \
  --image us-west1-docker.pkg.dev/ut-gee-wri-hydro-dev/utah-water/web:dev \
  --region us-west1 \
  --allow-unauthenticated

# 3. URL
gcloud run services describe utah-water-web --region us-west1 \
  --format 'value(status.url)'
```

## Notes

- Build context upload respects `.gitignore` (no `.gcloudignore` present), so
  `node_modules/` and stale `dist/` are never uploaded; the image is rebuilt
  from source each time.
- `--allow-unauthenticated` gives a shareable dev URL for a public-data
  dashboard. To restrict: redeploy with `--no-allow-unauthenticated` (callers
  then need IAM or IAP).
- Production images must be digest-pinned per repo convention
  (`infra/runtime/README.md`); this dev pipeline pins by tag only.
