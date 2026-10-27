# home-ops

GitOps configuration for the single-node `glumserver` k3s cluster. Argo CD
tracks the `main` branch of this repository and reconciles the Applications
defined in [`apps/nuc-prod`](./apps/nuc-prod).

This repository contains Kubernetes and Argo CD configuration, not
application state. Persistent data, most runtime credentials, DNS, the host
operating system, and the physical Zigbee coordinator must be backed up or
provisioned separately. PostgreSQL credentials are represented by an
encrypted SealedSecret.

## Cluster at a glance

The current host is `glumserver`, running Ubuntu 24.04 and k3s
`v1.35.5+k3s1` as a single node. Its current LAN address is `192.168.1.226`.
The cluster uses the k3s-bundled Traefik ingress controller, ServiceLB, and
`local-path` StorageClass; the manifests expect these defaults to remain
enabled.

Argo CD `v2.14.10` is installed from the pinned upstream manifest in
[`argocd/bootstrap/core`](./argocd/bootstrap/core). The root Application is
bootstrapped from [`argocd/bootstrap/apps`](./argocd/bootstrap/apps), then
tracks `apps/nuc-prod` on `main`. Root and child Applications use automated
sync, pruning, and self-healing.

| Workload | Current access |
| --- | --- |
| Argo CD | `http://glumserver.localdomain/argocd` |
| Home Assistant | `http://homeassistant.localdomain/` |
| ESPHome | NodePort `30052` (service port `6052`) |
| Matter Server | `ws://glumserver.localdomain/ws`; host networking also exposes port `5580` |
| Mosquitto MQTT | TCP `1883`; WebSocket `9001` |
| Node-RED | `http://glumserver.localdomain/nodered` |
| PostgreSQL | TCP `5432` |
| NetDaemon | Background automation; no Service or Ingress |
| Whisper | TCP `10300`; intentionally scaled to zero because it is not in use |
| Zigbee2MQTT | NodePort `30080` (service port `8080`) |

The ingress and LoadBalancer/NodePort services are defined in the manifests.
The hostnames and current LAN address are not: configure local DNS or hosts
entries for `glumserver.localdomain` and `homeassistant.localdomain`, and use a
DHCP reservation or update the network configuration if the node address
changes. Review router and firewall rules before exposing any service outside
the trusted LAN. The current ingress uses HTTP, not TLS.

## Repository layout

- [`apps/nuc-prod`](./apps/nuc-prod) — root app-of-apps list and child
  Application definitions.
- [`argocd/bootstrap`](./argocd/bootstrap) — one-time Argo CD installation,
  project, root Application, ingress, and middleware.
- [`base`](./base) — shared workload resources.
- [`overlays/nuc-prod`](./overlays/nuc-prod) — node-specific settings,
  PV/PVCs, ingress, hardware access, and deployment patches.
- [`docs`](./docs) — supporting documentation.

Argo CD owns the Kubernetes resources after bootstrap. Apply workload
manifests through Git and Argo CD; do not independently apply an overlay that
Argo CD is already managing.

## Host storage and data recovery

The PVs use local `hostPath` directories on the single node. Their paths are
absolute in [`overlays/nuc-prod/infra`](./overlays/nuc-prod/infra); preserve
these paths on a replacement host or update the PV manifests before syncing.
The capacities below are the current PV sizes, not a backup or a guarantee of
available disk space.

| Workload | Host path | PV capacity |
| --- | --- | --- |
| Home Assistant | `/home/epaulsen/containers/hass/volumes/ha-config` | 20 Gi |
| ESPHome | `/home/epaulsen/containers/hass/volumes/esphome` | 4 Gi |
| Matter Server | `/home/epaulsen/containers/hass/volumes/matter` | 2 Gi |
| Mosquitto | `/home/epaulsen/containers/hass/volumes/mosquitto-data` | 2 Gi |
| NetDaemon | `/home/epaulsen/containers/hass/volumes/netdaemon-config` | 1 Gi |
| Node-RED | `/home/epaulsen/containers/hass/volumes/node-red-data` | 2 Gi |
| PostgreSQL | `/mnt/ext_hdd/postgres-data` | 40 Gi |
| Whisper | `/home/epaulsen/containers/hass/volumes/whisper-data` | 8 Gi |
| Zigbee2MQTT | `/home/epaulsen/containers/hass/volumes/zigbee2mqtt-data` | 2 Gi |

Restore application data from a separate, verified backup before starting the
workloads. Home Assistant's configuration is a separate checkout of
[`epaulsen/ha-config`](https://github.com/epaulsen/ha-config) at
`/home/epaulsen/containers/hass/volumes/ha-config`; the other volume contents
are not stored in this repository. Preserve ownership and permissions during
restore. The PostgreSQL disk must be mounted at `/mnt/ext_hdd` before k3s and
the PostgreSQL workload start.

All PVs use the `Retain` reclaim policy. This prevents Kubernetes from
reclaiming the backing data when a claim is removed; it is not a backup.

Zigbee2MQTT also requires the physical coordinator to be attached at
`/dev/ttyACM0`, with `/dev/serial/by-id` present on the node. Its Deployment
selects the node labelled `hardware.zigbee=true`; identify the actual node
name after installing k3s and apply that label.

## Secrets and recovery material

Do not put secret values, private keys, or backup files in this repository.
Before bootstrapping the root Application, create the `home-ops` namespace and
provision these Secrets from a secure password manager or encrypted backup:

| Secret | Required keys | Used by |
| --- | --- | --- |
| `mqtt-credentials` | `username`, `password` | Mosquitto, Zigbee2MQTT, NetDaemon |
| `netdaemon-credentials` | `HASS_TOKEN`, `MQTT_PASSWORD`, `UnifiApiKey` | NetDaemon |
| `node-red-admin-auth` | `password-hash` | Node-RED |
| `postgres-secret` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | PostgreSQL |

PostgreSQL's `postgres-secret` is generated by the
[`SealedSecret`](./overlays/nuc-prod/postgres/sealed-secret.yaml). The current
cluster runs Sealed Secrets controller `0.37.0` (Helm chart `2.18.6`) in
`kube-system`; the controller is installed separately from this repository.
The controller's private sealing keys are required to decrypt the existing
SealedSecret. Keep an encrypted, off-repository backup of all active keys:

```bash
umask 077
kubectl -n kube-system get secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > /secure/backup/sealed-secrets-keys.yaml
```

During recovery, install the same controller chart, restore the backed-up key
Secrets into `kube-system`, restart the controller, and confirm the PostgreSQL
SealedSecret becomes Ready before syncing the PostgreSQL Application. If the
private keys are unavailable, recover the database credentials securely and
create a replacement SealedSecret with `kubeseal`; the encrypted manifest in
Git cannot be decrypted without a matching key.

The other three Secrets are not generated by these manifests. Create them in
`home-ops` before applying the root Application. For example, keep files
outside the repository with mode `0600`, containing only the corresponding
key/value pairs, then apply them with:

```bash
kubectl -n home-ops create secret generic mqtt-credentials \
  --from-env-file=/secure/backup/mqtt.env \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n home-ops create secret generic netdaemon-credentials \
  --from-env-file=/secure/backup/netdaemon.env \
  --dry-run=client -o yaml | kubectl apply -f -
```

`mqtt.env` must contain `username` and `password`. `netdaemon.env` must contain
`HASS_TOKEN`, `MQTT_PASSWORD`, and `UnifiApiKey`. Generate a Node-RED bcrypt
hash with the same image version used by the Deployment:

```bash
docker run --rm -it --entrypoint node-red nodered/node-red:4.1.3 admin hash-pw
```

Store the resulting hash securely and create `node-red-admin-auth` with the
`password-hash` key. The Argo CD initial admin password is generated by
Argo CD; retrieve it from the `argocd-initial-admin-secret` after bootstrap
and change it as part of securing a replacement cluster.

## Recreate the cluster

These steps rebuild the Kubernetes control plane and workloads. They do not
replace backups of application data, credentials, DNS, or the Sealed Secrets
private keys.

1. **Prepare the host.** Install a supported Ubuntu host, reserve its LAN
   address, configure local DNS for the hostnames above, mount the PostgreSQL
   disk at `/mnt/ext_hdd`, and restore all host-path data. Clone the separate
   Home Assistant configuration repository into its expected path:

   ```bash
   git clone git@github.com:epaulsen/ha-config.git \
     /home/epaulsen/containers/hass/volumes/ha-config
   ```

   Attach the Zigbee coordinator and confirm `/dev/ttyACM0` and
   `/dev/serial/by-id` exist.
2. **Install k3s.** The current cluster runs `v1.35.5+k3s1`. Keep the default
   Traefik, ServiceLB, and local-path components enabled:

   ```bash
   curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.35.5+k3s1" sh -
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   kubectl get nodes
   ```

   Label the node that has the Zigbee coordinator:

   ```bash
   kubectl label node <node-name> hardware.zigbee=true --overwrite
   ```

3. **Install Sealed Secrets and restore its keys.** Install [Helm
   3](https://helm.sh/docs/intro/install/), then install
   the current chart release and restore the secure key backup before starting
   GitOps:

   ```bash
   helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
   helm repo update
   helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets \
     --namespace kube-system --create-namespace --version 2.18.6

   kubectl -n kube-system apply -f /secure/backup/sealed-secrets-keys.yaml
   kubectl -n kube-system rollout restart deployment/sealed-secrets
   kubectl -n kube-system rollout status deployment/sealed-secrets
   kubectl wait --for=condition=Established \
     crd/sealedsecrets.bitnami.com --timeout=120s
   ```

4. **Prepare namespace and credentials.** Create the application namespace
   and the `mqtt-credentials`, `netdaemon-credentials`, and
   `node-red-admin-auth` Secrets described above. Do this before creating the
   root Application so workloads do not start without their required
   credentials:

   ```bash
   kubectl create namespace home-ops --dry-run=client -o yaml | kubectl apply -f -

   read -rsp 'Node-RED bcrypt hash: ' NODE_RED_ADMIN_PASSWORD_HASH
   printf '\n'
   kubectl -n home-ops create secret generic node-red-admin-auth \
     --from-literal=password-hash="$NODE_RED_ADMIN_PASSWORD_HASH" \
     --dry-run=client -o yaml | kubectl apply -f -
   unset NODE_RED_ADMIN_PASSWORD_HASH
   ```

   Create `mqtt-credentials` and `netdaemon-credentials` from the protected
   env files shown above. The PostgreSQL `SealedSecret` is applied by Argo CD;
   the restored controller key must be available before that sync.
5. **Install Argo CD.** Wait for the k3s Traefik CRDs to be established, then
   apply the pinned Argo CD installation and wait for its server:

   ```bash
   kubectl wait --for=condition=Established \
     crd/middlewares.traefik.io --timeout=120s
   kubectl apply -k argocd/bootstrap/core
   kubectl -n argocd rollout status deployment/argocd-server --timeout=180s
   ```

6. **Start GitOps.** Apply the project and root Application:

   ```bash
   kubectl apply -k argocd/bootstrap/apps
   ```

   Argo CD then creates the child Applications from `apps/nuc-prod` and
   reconciles the cluster from `main`. Do not manually apply the workload
   overlays.
7. **Verify recovery.** Check that the node is Ready, all Applications are
   `Synced` and `Healthy`, all expected PVCs are `Bound`, and workloads are
   Ready. Confirm that the PostgreSQL `SealedSecret` is Ready and its
   `postgres-secret` exists. Whisper is intentionally at zero replicas.
   Confirm ingress, service ports, PostgreSQL data, and application data
   before considering recovery complete.

Useful checks:

```bash
kubectl get nodes
kubectl -n argocd get applications
kubectl -n home-ops get pods,pvc
kubectl get pv
kubectl get ingress -A
```
