
### ✅ Phase 12 — Paperless-ngx

Deployed via raw manifests (CrystalNET Helm chart abandoned due to
Bitnami image issues and hardcoded security contexts).

Components:
- paperless-web (acer-8gb) — main app
- paperless-postgres (asus-4gb) — database
- paperless-redis (asus-4gb) — cache

Gotchas:
- Service named "paperless" causes env var conflict with k8s
  injecting PAPERLESS_PORT — rename service to "paperless-web"
- OCR language "ron" not pre-installed — use PAPERLESS_OCR_LANGUAGES
  env var to install it, or remove from PAPERLESS_OCR_LANGUAGE
- Newt must be on same node as paperless for service routing to work
  (asus-8gb has monolithic kernel, no loadable modules)
