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

Argo CD ingress host in this scaffold: `argocd.home-ops.local`.

## 3) Workload topology

- Child Applications live under `k8s/apps/nuc-prod`.
- Base manifests: `k8s/base/<service>`.
- NUC-specific overlays: `k8s/overlays/nuc-prod/<service>`.

## 4) Data re-use from existing docker-compose folders

Overlay creates static hostPath PVs so K8s reuses existing on-disk state:

- Postgres: `/mnt/ext_hdd/postgres-data`
- MQTT: `/home/epaulsen/containers/hass/mosquitto-data`
- Zigbee2MQTT: `/home/epaulsen/containers/hass/zigbee2mqtt-data`
- Whisper: `/home/epaulsen/containers/hass/whisper-data`
- ESPHome: `/home/epaulsen/containers/hass/esphome`

Adjust paths if your compose project directory on NUC is different.

## 5) Zigbee2MQTT device mapping on NUC

This scaffold mounts both:

- `/dev/ttyACM0` (current compose behavior)
- `/dev/serial/by-id` (stable path source for config)

Recommended: set Zigbee serial port in Zigbee2MQTT config to `/dev/serial/by-id/<your-stick-id>`, then keep udev naming stable on host.
