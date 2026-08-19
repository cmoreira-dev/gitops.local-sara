# gitops.local-sara

GitOps deployment for **Sara** — see `ui.ia.local-sara`/`api.ia.local-sara` for the
application itself. Two Applications from one repo (like `gitops.teupadel.com`, not the
one-app-per-repo `gitops.template` default — see below for why).

```
gitops.local-sara/
├── argocd/
│   ├── sara-api.yaml   ← Application → helm/api
│   └── sara-ui.yaml    ← Application → helm/ui
└── helm/
    ├── api/             ← generic-app wrapper chart, values for api.ia.local-sara
    └── ui/               ← generic-app wrapper chart, values for ui.ia.local-sara
```

## Why `argocd/` isn't empty

`gitops.template`'s convention is an empty `argocd/` (just `.gitkeep`) because the
`ApplicationSet/gitops-repos` auto-generates one Application per `gitops.*` repo,
pointing at a single `helm/` directory. Sara ships two independently-deployed
components (api + ui, separate images/Services/routes) from one repo, which is exactly
the case the template's own README calls out as needing manual `argocd/` population.

## Routing

Both apps share the existing `local.cmoreira.dev` hostname (Gateway
`nginx-gateway-cmoreira-dev`, already tunneled — no changes needed in
`gitops.core-addons`) under different paths:

- UI: `local.cmoreira.dev/sara` — prefix preserved end-to-end; the Next.js app has
  `basePath: "/sara"` baked in at build time and expects it.
- API: `local.cmoreira.dev/sara/api` — prefix **stripped** by the HTTPRoute's
  `URLRewrite` filter before hitting the container, so `api.ia.local-sara`'s routes stay
  plain (`/health`, `/search`, ...). `/sara/api` wins over the UI's `/sara` catch-all on
  the same hostname via Gateway API's longest-prefix-match — no extra config needed for
  that part.

## No ExternalSecrets

Neither app needs one — `api.ia.local-sara` is a public Cifra Club scraper with no API
keys, and `ui.ia.local-sara`'s build-time config (`NEXT_PUBLIC_API_URL`, `NEXT_BASE_PATH`)
isn't a runtime secret either. See each `helm/*/values.yaml` for the one-line note.

## Local dependency resolution

```bash
helm dependency update helm/api
helm dependency update helm/ui
```

Generates `Chart.lock` + `charts/` for each (committed) — same OCI dependency
(`oci://ghcr.io/cmoreira-dev/charts`, not the `gitops.template` default's
`git+https://...`, which needs a `helm-git` ArgoCD repo-server plugin that isn't
configured yet; teupadel already proves the OCI path works in this cluster).
