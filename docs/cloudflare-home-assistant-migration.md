# Home Assistant Migration and Remote Access Plan

## Purpose

Move Home Assistant, Node-RED, Grafana, and VS Code Server from the
VirtualBox Home Assistant installation into the existing single-node k3s
cluster. Keep Home Assistant remote access on Nabu Casa. Publish only
administration applications through Cloudflare Tunnel and Cloudflare Access,
without opening inbound firewall ports.

## Target Architecture

```text
                    Home Assistant Cloud (Nabu Casa)
                              |
                              v
                        Home Assistant
                       k3s: home-ops
                              |
      +-----------------------+------------------------+
      |                       |                        |
      v                       v                        v
    MQTT                  PostgreSQL                Whisper

Internet
   |
   v
Cloudflare Access
  Google identity provider
   |
   v
Cloudflare Tunnel
  outbound-only connection from k3s
   |
   v
Traefik (k3s ingress controller)
   |
   v
apps.mydomain.com
  /node-red  -> Node-RED
  /grafana   -> Grafana
  /code      -> code-server
  /argocd    -> Argo CD (optional)
```

There are no inbound router or firewall rules. `cloudflared` establishes an
outbound encrypted connection to Cloudflare. Cloudflare sends matching public
requests through that tunnel to Traefik.

## Public URLs and Authentication

| URL | Application | Authentication |
|---|---|---|
| Nabu Casa Remote UI URL | Home Assistant | Home Assistant authentication and Nabu Casa |
| `https://apps.mydomain.com/node-red` | Node-RED editor | Cloudflare Access with Google, plus Node-RED admin authentication |
| `https://apps.mydomain.com/grafana` | Grafana | Cloudflare Access with Google, plus Grafana authentication or OIDC |
| `https://apps.mydomain.com/code` | VS Code Server | Cloudflare Access with Google, plus code-server authentication initially |
| `https://apps.mydomain.com/argocd` | Argo CD, optional | Cloudflare Access with Google, plus Argo CD authentication |

Home Assistant is intentionally not configured as a Cloudflare Tunnel public
hostname. Nabu Casa remains its sole remote route.

Cloudflare Access is an outer security layer, not replacement for each
application's native authentication. Keep application authentication enabled
until direct-origin bypass has been proven impossible and access policies are
reviewed.

## Home Assistant Container Model

Use the `ghcr.io/home-assistant/home-assistant` container. Home Assistant
Container does not support Home Assistant add-ons. Existing Node-RED, Grafana,
and VS Code Server add-ons become independent Kubernetes workloads.

This separation has benefits:

- Application updates and failures are isolated from Home Assistant.
- Each application has its own resource limits, persistence, ingress, and
  security settings.
- Argo CD and Kustomize own desired state rather than Supervisor add-on
  configuration.

### Home Assistant Storage

Persist `/config` on a static hostPath PV, following current repository
conventions:

```text
Host path:      /home/epaulsen/containers/hass/homeassistant-config
Container path: /config
```

This keeps HA YAML accessible through SSH from local VS Code. Home Assistant
must run one replica only and use `Recreate` deployment strategy. Two running
instances must never write same `/config` directory.

Use PostgreSQL for HA recorder/history instead of SQLite where possible. Keep
configuration, custom components, automations, secrets, and media paths on
the HA config volume as appropriate.

HA configuration must trust Traefik only if it is later accessed through an
HTTP reverse proxy:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    # Exact Traefik source CIDR or pod IP range only.
```

Nabu Casa does not require publishing this ingress. Do not add broad trusted
proxy ranges without identifying actual Traefik source addresses.

### Kubernetes Layout

Add these paths, matching current `base` and `overlays/nuc-prod` structure:

```text
k8s/base/homeassistant/
  deployment.yaml
  service.yaml
  pvc.yaml
  kustomization.yaml

k8s/overlays/nuc-prod/homeassistant/
  patch-deployment.yaml
  patch-pvc.yaml
  kustomization.yaml

k8s/overlays/nuc-prod/infra/
  homeassistant-pv.yaml
```

Add an Argo CD Application under `k8s/apps/nuc-prod/` and include it in that
directory's `kustomization.yaml`.

## Application Deployments

All three workloads should be single replica and receive independent PVCs.
Use immutable or versioned image tags after testing; avoid floating `latest`
tags.

### Node-RED

Persist `/data`, including flows, credentials, installed nodes, and
`settings.js`.

Configure subpath operation in `settings.js`:

```js
httpAdminRoot: "/node-red",
httpNodeRoot: "/node-red/api",
```

Traefik must route `PathPrefix("/node-red")` to Node-RED without stripping
the prefix. Node-RED's editor uses WebSockets; Traefik supports upgrade
requests without special middleware.

Enable Node-RED `adminAuth` even though Cloudflare Access authenticates first.
Use a bcrypt password hash in a Kubernetes Secret, not Git.

Review every HTTP In flow before public exposure. Put public flow endpoints
under `/node-red/api` deliberately; do not unintentionally expose endpoints at
the root domain.

### Grafana

Persist `/var/lib/grafana`. Provision dashboards and data sources through
versioned Kubernetes ConfigMaps or mounted files where practical, while
preserving generated data in its PVC.

Configure `grafana.ini`:

```ini
[server]
root_url = https://apps.mydomain.com/grafana/
serve_from_sub_path = true

[security]
allow_embedding = true
```

`allow_embedding` is required only if Grafana will appear in an HA sidebar
iframe. It weakens browser clickjacking protection, so use it only with
Cloudflare Access and native Grafana authentication still enabled.

Grafana can use Google OAuth directly or an OIDC provider. Cloudflare Access
remains common outer gate regardless.

### VS Code Server

Run `coder/code-server` as independent deployment. Use two storage mounts:

```text
HA config PV                -> /home/coder/project
code-server configuration   -> /home/coder/.local/share/code-server
```

Mounting HA's config PV into code-server makes remote editing operate on exact
same files that HA uses. The separate code-server PVC preserves editor
extensions and settings without placing them in HA configuration.

Retain code-server password authentication during initial rollout. Only
consider disabling it after confirming every ingress path is exclusively
reachable through Cloudflare Access.

## Cloudflare Configuration

### Domain and DNS

1. Register domain.
2. Add domain to Cloudflare.
3. Change registrar nameservers to Cloudflare-assigned nameservers.
4. Create `apps.mydomain.com` as Cloudflare Tunnel public hostname.
5. Do not create an equivalent tunnel hostname for Home Assistant.

### Tunnel

Create one named tunnel in Cloudflare Zero Trust. Store its token in a
Kubernetes Secret. Deploy `cloudflared` in k3s in `home-ops` or dedicated
namespace.

Tunnel target:

```text
Public hostname: apps.mydomain.com
Service:         http://traefik.kube-system.svc.cluster.local:80
```

Traefik selects backend based on preserved `Host` header and URL path. Tunnel
traffic stays inside cluster after it reaches `cloudflared`.

Use token-based tunnel deployment initially. Do not commit token, account
credentials, Google OAuth credentials, or Cloudflare API tokens to Git.
Use Kubernetes Secrets, ideally encrypted through current Sealed Secrets
workflow.

### Cloudflare Access

Configure a Self-hosted Access application:

```text
Application domain: apps.mydomain.com
Path:               /*
Identity provider:   Google
Policy:              allow explicitly listed Google account email addresses
Session duration:    short, appropriate for household use
```

Add Google as Cloudflare Access identity provider. Restrict policy to personal
account(s), not all Google users. Add separate tighter policy for `/argocd`
if that route is published.

Cloudflare Access supports WebSockets needed by Node-RED and code-server.

## Traefik Routes

Create HTTPS-facing Ingress resources for host `apps.mydomain.com`:

| Path | Backend service | Prefix handling |
|---|---|---|
| `/node-red` | `node-red` | Preserve prefix |
| `/grafana` | `grafana` | Preserve prefix |
| `/code` | `code-server` | Configure code-server base path and preserve prefix |
| `/argocd` | `argocd-server` | Requires Argo CD root path/base href configuration |

Cloudflare provides edge TLS for browser-to-Cloudflare traffic. Tunnel uses
encrypted Cloudflare connection. Internal Traefik connection may start as HTTP
inside cluster; assess internal TLS only if cluster threat model requires it.

Current repo uses `glumserver.localdomain` and Traefik `web` entrypoint for
ESPHome and Argo CD. New public routes should use `apps.mydomain.com` and
must not replace existing LAN host routes until migration completes.

## Home Assistant Sidebar Links

HA supports iframe panels:

```yaml
panel_iframe:
  node_red:
    title: Node-RED
    icon: mdi:shuffle-variant
    url: https://apps.mydomain.com/node-red
    require_admin: true
```

Repeat for Grafana if embedding is enabled.

`require_admin` only controls HA sidebar visibility. It does not protect
public URL; Cloudflare Access and application authentication do that.

Browser third-party-cookie restrictions can prevent Cloudflare Access or
application session login inside iframe hosted from Nabu Casa domain. Use
direct bookmarks to `apps.mydomain.com` as reliable primary access path.

## Migration Procedure

### 1. Prepare and Back Up

1. Back up HA configuration, Node-RED `/data`, Grafana data/provisioning, and
   VS Code Server data.
2. Export HA automations, dashboards, integrations, and secrets as needed.
3. Record add-on settings, environment variables, network ports, mounted
   paths, and installed Node-RED nodes.
4. Verify current Postgres, MQTT, Zigbee2MQTT, and Whisper endpoints.
5. Plan downtime window; prevent concurrent edits while copying state.

### 2. Prepare Cluster Storage

1. Create host directories with ownership compatible with each container.
2. Add static hostPath PVs and matching PVCs.
3. Verify mounted directory permissions using temporary workload or init
   container.
4. Ensure HA config directory is visible through host SSH as planned.
5. Keep separate writable storage for each application; share only HA config
   PV with code-server.

### 3. Deploy and Test Internal Workloads

1. Deploy HA with a `ClusterIP` Service.
2. Configure HA to use cluster MQTT, Postgres, and Whisper service names.
3. Deploy Node-RED, Grafana, and code-server with `ClusterIP` Services.
4. Import application data one workload at a time.
5. Test workloads locally through temporary port-forward or existing internal
   LAN routing before external publication.
6. Verify HA recorder database, MQTT integration, Zigbee entity availability,
   backups, and Whisper integration.

### 4. Deploy Tunnel and Public Routes

1. Deploy `cloudflared` using Secret-provided tunnel token.
2. Create `apps.mydomain.com` Traefik Ingress paths.
3. Configure Cloudflare Access Google identity provider and allow policy.
4. Publish Node-RED first and test Google authentication, WebSockets, and
   Node-RED native login.
5. Publish Grafana and code-server after subpath and persistence checks.
6. Keep Argo CD unpublished unless needed; it has highest administrative
   impact.

### 5. Cut Over

1. Stop matching VM add-on before final data copy.
2. Copy final config/data changes into k3s volumes.
3. Start k3s workload and confirm expected behavior.
4. Point Nabu Casa integration at cluster-hosted HA only after local HA works.
5. Keep VM powered off but recoverable through observation period.
6. Remove VirtualBox VM only after backups and rollback decision are accepted.

## Security Controls

- No inbound router port forwarding.
- No publicly exposed PostgreSQL.
- Convert Postgres, MQTT, and Whisper Services from `LoadBalancer` to
  `ClusterIP` if no LAN clients require them. Confirm before changing existing
  service exposure.
- Separate Kubernetes Secrets for database credentials, tunnel token,
  Node-RED password hash, Grafana admin password, and code-server password.
- Encrypt Git-managed Secrets with Sealed Secrets; never commit plaintext.
- Restrict Cloudflare Access policy to known Google identities.
- Keep application-level auth enabled.
- Use strong unique credentials and MFA on Google account and Cloudflare
  account.
- Back up hostPath data outside NUC before destructive upgrades.
- Keep Argo CD behind separate restrictive Access policy or avoid public route.

## Rollback

Keep VirtualBox VM and its data intact until k3s deployment is stable.

If HA migration fails:

1. Stop k3s Home Assistant workload.
2. Restore VM's HA configuration and start VM.
3. Do not allow both HA instances to connect to same MQTT automations or write
   same database/configuration storage.
4. Diagnose using copied data, not original VM files.

If public app access fails, tunnel/ingress failure does not affect HA local
operation or Nabu Casa remote access. Disable affected Cloudflare public
hostname or Access application while keeping private workload available.

## Implementation Checklist

- [ ] Register domain and configure Cloudflare DNS.
- [ ] Create Cloudflare Zero Trust organization and Google identity provider.
- [ ] Create Cloudflare Tunnel and encrypted Kubernetes Secret.
- [ ] Add `cloudflared` deployment and service account as needed.
- [ ] Add Home Assistant base, overlay, PV, PVC, service, and Argo CD app.
- [ ] Add Node-RED base, overlay, PV, PVC, service, config, and Argo CD app.
- [ ] Add Grafana base, overlay, PV, PVC, service, config, and Argo CD app.
- [ ] Add code-server base, overlay, PV, PVC, service, config, and Argo CD app.
- [ ] Add `apps.mydomain.com` Traefik Ingress resources.
- [ ] Configure application subpaths and native credentials.
- [ ] Back up VM data and perform staged migration.
- [ ] Test Nabu Casa access, Cloudflare Access, persistence, WebSockets, and
  restart recovery.
- [ ] Retire VM only after stable observation period.
