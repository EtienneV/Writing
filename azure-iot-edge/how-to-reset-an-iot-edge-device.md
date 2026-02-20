# How to Reset an IoT Edge Device

*A step-by-step guide to cleanly wiping an Azure IoT Edge device and returning it to a pristine state.*

---

Whether you're decommissioning a device, handing it off to another team, or simply troubleshooting a broken deployment, knowing how to fully reset an Azure IoT Edge device is an essential skill for any IoT engineer. In this article we'll walk through every step needed to stop all running modules, remove the IoT Edge runtime configuration, and optionally re-provision the device from scratch.

---

## Why Would You Need to Reset a Device?

There are several common situations where a full reset is the right move:

- **Re-provisioning** — You want to register the device under a different IoT Hub or use a new device identity.
- **Troubleshooting** — A corrupt configuration or a misbehaving module has left the runtime in an unrecoverable state.
- **Decommissioning** — The device is being retired or re-purposed and must not retain any secrets or configuration.
- **Testing** — You need a clean baseline to validate a fresh installation procedure.

---

## Prerequisites

- An Azure IoT Edge device running Linux (Ubuntu 18.04 / 20.04 / 22.04 or compatible).
- SSH access (or physical console access) to the device.
- `sudo` privileges on the device.

> **Note:** The commands below target the **IoT Edge 1.4 (LTS)** release which uses the `aziot-edge` / `aziot-identity-service` package set. For older 1.1 LTS installations that still use `iotedge` + `libiothsm-std`, the package names differ slightly — see the [migration guide](https://docs.microsoft.com/azure/iot-edge/how-to-update-iot-edge) for details.

---

## Step 1 — Stop the IoT Edge Runtime

Before touching any files, stop all running services gracefully.

```bash
sudo iotedge system stop
```

This command stops `aziot-edged` and all associated Azure IoT identity services. You can confirm everything is down with:

```bash
sudo iotedge system status
```

You should see all services report `stopped` or `inactive`.

---

## Step 2 — Remove All Running Modules and Their Data

IoT Edge modules are Docker/Moby containers. Even after the runtime stops, their containers and images may still be present on disk.

### List all containers (including stopped ones)

```bash
sudo docker ps -a
```

### Remove every container

```bash
sudo docker rm -f $(sudo docker ps -aq)
```

### Remove all module images (optional but recommended for a clean slate)

```bash
sudo docker rmi -f $(sudo docker images -q)
```

> **Tip:** If you plan to re-deploy the same modules right away you can skip the image removal step — Docker will reuse cached layers and the deployment will be faster.

---

## Step 3 — Wipe the IoT Edge Configuration

The `aziot-edge` package stores its configuration and certificates under `/etc/aziot/` and runtime state under `/var/lib/aziot/`. Remove both:

```bash
sudo rm -rf /etc/aziot/
sudo rm -rf /var/lib/aziot/
```

If you are on the older **1.1 LTS** release, the paths are different:

```bash
# IoT Edge 1.1 only
sudo rm -f /etc/iotedge/config.yaml
sudo rm -rf /var/lib/iotedge/
sudo rm -rf /var/lib/libiothsm/
```

---

## Step 4 — Reprovision the Runtime (Optional)

If you just want to apply a new configuration without reinstalling the package, run the built-in reprovision command after placing a fresh `config.toml` file in `/etc/aziot/`:

```bash
# Copy and edit the sample configuration
sudo cp /etc/aziot/config.toml.edge.template /etc/aziot/config.toml
sudo nano /etc/aziot/config.toml
```

Set your provisioning method (manual connection string, DPS symmetric key, DPS X.509, etc.), then apply:

```bash
sudo iotedge config apply
```

Finally, restart the runtime:

```bash
sudo iotedge system restart
```

---

## Step 5 — Verify the Reset

Once the runtime is back up (or if you chose not to reprovision), confirm the device is in the expected state:

```bash
# Check service health
sudo iotedge system status

# Check running modules
sudo iotedge list

# View the last 50 log lines of the edge agent
sudo iotedge logs edgeAgent --tail 50
```

After a clean reset with no new configuration, `iotedge list` should return an empty list. After reprovisioning, you should see `edgeAgent` running within a minute or two as the cloud-side deployment is pulled down.

---

## Full Reset Script

If you need to automate the process, here is a minimal shell script that combines all the steps above:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Stopping IoT Edge runtime..."
sudo iotedge system stop

echo "Removing all Docker containers..."
CONTAINERS=$(sudo docker ps -aq)
if [ -n "$CONTAINERS" ]; then
  sudo docker rm -f "$CONTAINERS"
fi

echo "Wiping IoT Edge configuration and state..."
sudo rm -rf /etc/aziot/
sudo rm -rf /var/lib/aziot/

echo "Done. The device has been reset."
echo "Place a new /etc/aziot/config.toml and run 'sudo iotedge config apply' to reprovision."
```

Save it as `reset-iotedge.sh`, make it executable with `chmod +x reset-iotedge.sh`, and run it with `sudo`.

---

## Conclusion

Resetting an Azure IoT Edge device is a straightforward five-step process: stop the runtime, remove all module containers, delete the configuration and state directories, and optionally reprovision with a new configuration. Having this procedure documented — and better yet scripted — makes device lifecycle management significantly less stressful, whether you're running a handful of devices in the lab or thousands in the field.

---

*Found this useful? Follow along for more Azure IoT Edge deep-dives.*
