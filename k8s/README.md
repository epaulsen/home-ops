# home-ops Kubernetes layout

This repository now contains a Kustomize + Argo CD GitOps layout for a single-node k3s NUC.

## 1) Bootstrap k3s on NUC

```bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
sudo kubectl label node <nuc-node-name> hardware.zigbee=true
```

## 2) Bootstrap Argo CD

```bash
kubectl apply -k k8s/argocd/bootstrap/core
kubectl -n argocd rollout status deploy/argocd-server --timeout=180s
kubectl apply -k k8s/argocd/bootstrap/apps
```

Argo CD is accessible at `http://glumserver.localdomain/argocd`.

## 3) Workload topology

- Child Applications live under `k8s/apps/nuc-prod`.
- Base manifests: `k8s/base/<service>`.
- NUC-specific overlays: `k8s/overlays/nuc-prod/<service>`.

## 4) Data re-use from existing docker-compose folders

Overlay creates static hostPath PVs so K8s reuses existing on-disk state:

- Postgres: `/mnt/ext_hdd/postgres-data`
- MQTT: `/home/epaulsen/containers/hass/mosquitto-data`
- Zigbee2MQTT: `/home/epaulsen/containers/hass/zigbee2mqtt-data`
- Node-RED: `/home/epaulsen/containers/hass/node-red-data`
- Whisper: `/home/epaulsen/containers/hass/whisper-data`
- ESPHome: `/home/epaulsen/containers/hass/esphome`

Adjust paths if your compose project directory on NUC is different.

## 5) Zigbee2MQTT device mapping on NUC

This scaffold mounts both:

- `/dev/ttyACM0` (current compose behavior)
- `/dev/serial/by-id` (stable path source for config)

Recommended: set Zigbee serial port in Zigbee2MQTT config to `/dev/serial/by-id/<your-stick-id>`, then keep udev naming stable on host.

## 6) Node-RED migration

Create the HA long-lived access-token Secret in `home-ops` before syncing the Node-RED Argo CD application. The token is not stored in Git:

```bash
read -rsp 'Long-lived access token: ' HA_TOKEN
printf '\n'
kubectl -n home-ops create secret generic node-red-credentials \
	--from-literal=ha-token="$HA_TOKEN" \
	--dry-run=client -o yaml | kubectl apply -f -
unset HA_TOKEN
```

The deployment exposes this value as `HA_TOKEN`. After importing the flow backup, update the Home Assistant server configuration in Node-RED to use the HA instance's network address and this token; the add-on's Supervisor connection is not available from Kubernetes.
