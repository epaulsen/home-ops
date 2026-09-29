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

After importing the flow backup, open a Home Assistant node and edit its shared server configuration. Disable the Home Assistant add-on option, set the Base URL to the HA instance's address reachable from the cluster (for example, `http://<ha-vm-ip>:8123`), and paste the long-lived access token into the Access Token field. Deploy the changes and verify the node reports a connection. The add-on's Supervisor connection is not available from Kubernetes.

The server configuration is stored in the persistent Node-RED `/config` volume; no Kubernetes Secret is required for the HA token.
