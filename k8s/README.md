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

Node-RED runs as a standalone container on port 1880 and is available only through the authenticated ingress at `http://glumserver.localdomain/nodered`. The `/data` directory is persistent on the NUC.

### Before Argo CD sync

Generate a bcrypt hash for the Node-RED editor password:

```bash
docker run --rm -it --entrypoint node-red nodered/node-red:4.1.3 admin hash-pw
```

Create the Kubernetes Secret using the hash printed by that command:

```bash
read -rsp 'Bcrypt password hash: ' NODE_RED_ADMIN_PASSWORD_HASH
printf '\n'
kubectl -n home-ops create secret generic node-red-admin-auth \
	--from-literal=password-hash="$NODE_RED_ADMIN_PASSWORD_HASH" \
	--dry-run=client -o yaml | kubectl apply -f -
unset NODE_RED_ADMIN_PASSWORD_HASH
```

### Cutover

1. Merge the PR, let Argo CD sync, then wait for `kubectl -n home-ops rollout status deploy/node-red` to complete.
2. Open Node-RED and install `node-red-contrib-home-assistant-websocket` from **Menu > Manage palette > Install**. The standalone image does not include the Home Assistant nodes; the persistent `/data` volume keeps the package installed.
3. Stop the old Node-RED app in Home Assistant Supervisor before importing or deploying the flows, so both instances do not run the same automations.
4. Import the flow backup. Edit the shared Home Assistant server configuration: disable the add-on option, set Base URL to the HA VM address reachable from the cluster (for example, `http://<ha-vm-ip>:8123`), and paste the long-lived HA token into Access Token.
5. Deploy and verify that the Home Assistant nodes connect and the expected flows run.
