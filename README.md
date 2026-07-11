# homelab_config

GitOps-style config for a single-node k3s cluster (`richardshomelab`), reached over Tailscale.

## apps/hello-site

A minimal static page served by nginx, deployed with plain `kubectl apply` (no registry, no Argo CD yet).

**Cluster/node facts:**
- Node: `richard@richardshomelab` (Tailscale IP `100.78.31.32`)
- No image registry — the image is built directly on the node and referenced with `imagePullPolicy: Never`
- Ingress: k3s's built-in Traefik, host `hello.homelab.lan`

### Redeploy after changing `apps/hello-site/html/`

1. Copy the app to the node and rebuild the image there:
   ```
   scp -r apps/hello-site richard@richardshomelab:~/hello-site
   ssh richard@richardshomelab 'cd ~/hello-site && docker build -t hello-site:latest .'
   ```
2. Import the image into k3s's containerd (separate from Docker's own store — needs sudo on the node):
   ```
   ssh richard@richardshomelab 'docker save hello-site:latest | sudo k3s ctr images import -'
   ```
3. Roll the deployment to pick up the new image:
   ```
   kubectl rollout restart deployment/hello-site -n default
   ```

### Access

Add this to `/etc/hosts` (Tailscale IP, so it works off the LAN too):
```
100.78.31.32 hello.homelab.lan
```
Then visit `http://hello.homelab.lan`.
