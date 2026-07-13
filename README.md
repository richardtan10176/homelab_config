# homelab_config

GitOps config for a single-node k3s cluster (`richardshomelab`), exposed to
the public internet via a Cloudflare Tunnel.

## Layout

- `apps/<project>/` — one folder per app (Dockerfile, static assets, `k8s/`
  manifests). `hello-site` and `landing` are the current examples.
- `infra/argocd/` — Argo CD's own config. `root-app.yaml` is an "app of
  apps": it watches `infra/argocd/apps/` and auto-creates whatever
  `Application` manifests it finds there.
- `infra/cloudflared/` — the tunnel that gets traffic from the public
  internet into the cluster.

## How a deploy happens

Push to `main` touching an app's `html/`/`Dockerfile` → GitHub Actions builds
the image and pushes it to GHCR → the workflow commits the new image tag
into that app's `k8s/deployment.yaml` → Argo CD notices the git change and
syncs it onto the cluster. No manual `kubectl` needed for day-to-day changes.

## Adding a new project

1. Copy `apps/hello-site/` to `apps/<name>/`, swap in your app's
   Dockerfile/content, and set `k8s/ingress.yaml`'s host to
   `<name>.r-tan.dev`.
2. Copy `.github/workflows/build-hello-site.yml` to
   `build-<name>.yml`, retargeting the paths/image name at `apps/<name>`.
3. Copy `infra/argocd/apps/hello-site.yaml` to `<name>.yaml`, pointing
   `source.path` at `apps/<name>/k8s`.
4. Push to `main` — Argo CD picks up the new `Application` automatically.
5. Add a link to it on `apps/landing/html/index.html`.

No DNS or tunnel changes needed — `infra/cloudflared` routes the wildcard
`*.r-tan.dev` to the cluster's ingress controller, which does per-app
routing by hostname.

## One-time cluster setup

Domain (`r-tan.dev`, registered on Namecheap, nameservers pointed at
Cloudflare), the Cloudflare Tunnel (`homelab-k3s`), and its DNS routes are
already set up. What's left, against the cluster itself:

1. Install Argo CD on the cluster (official install manifest), then apply
   this repo's bootstrap manifests once:
   ```
   kubectl apply -f infra/argocd/ingress.yaml
   kubectl apply -f infra/argocd/root-app.yaml
   ```
   Everything else (hello-site, landing, cloudflared) syncs automatically
   from there.
2. Create the tunnel credentials secret from the JSON file `cloudflared
   tunnel create` wrote locally — never commit this file:
   ```
   kubectl create namespace cloudflared --dry-run=client -o yaml | kubectl apply -f -
   kubectl create secret generic cloudflared-credentials \
     --from-file=credentials.json=<path-to-the-tunnel-json-file> \
     -n cloudflared
   ```
3. After the first CI run for each app, set its GHCR package visibility to
   public (avoids needing an `imagePullSecret` on the cluster).

### Access

- Public: `https://r-tan.dev` (landing page), `https://<app>.r-tan.dev`
- Argo CD UI stays private: add `100.78.31.32 argocd.homelab.lan` to
  `/etc/hosts` (Tailscale IP), then visit `http://argocd.homelab.lan`.
