# How to Reset an Azure IoT Edge Device

When working with Azure IoT Edge, you’ll eventually face a situation where your device gets into an inconsistent state, like:

- Modules not starting
- Edge Agent failing deployment
- Corrupted storage
- Certificate issues
- Strange behavior after reprovisioning

Instead of spending hours debugging a broken state, sometimes the cleanest solution is simple:

> 🔄 Reset the IoT Edge device and let it reprovision cleanly.

This article walks through a safe and practical way to reset an IoT Edge device:

- Removes all Edge modules and docker containers
- Delete Edge Hub and Edge Agent local storage
- Delete local certificates and keys

> ⚠️ Do **not** do this in production unless you understand the consequences.

# Step-by-Step Reset Procedure

## 1 - Make Sure the Device Has Internet Access

After reset, the device must reconnect to:

- Your IoT Hub
- Azure Container Registry (if used)
- DPS (if provisioning with DPS)

Without connectivity, reprovisioning will fail.

## 2 - Stop the IoT Edge Runtime

Before touching any files, stop all running services gracefully.

```bash
sudo iotedge system stop
```

This command stops `aziot-edged` and all associated Azure IoT identity services. You can confirm everything is down with:

```bash
sudo iotedge system status
```

## 3 - Remove all Docker containers

```bash
sudo docker rm -v -f $(sudo docker ps -qa)
```

This removes all IoT Edge modules, and any leftover containers

### Optional: Remove all Docker images

If necessary, you can even remove all Docker containers, networks, images and modules. IoT Edge will need to re-download all necessary images after reprovisioning.

```bash
sudo docker system prune --all --volumes
```

> **Tip:** If you plan to re-deploy the same modules right away you can skip this image removal step — Docker will reuse cached layers and the deployment will be faster. 

## 4 - Remove Edge Runtime Storage

If you configured the edgeAgent and edgeHub modules to persist their data on the host [like it is recommended](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-access-host-storage-from-module#configure-system-modules-to-use-persistent-storage), delete their storage folders. These folders are defined by the modules environment variables `storageFolder` and the bindings in the container create options (`HostConfig > Binds`)

```bash
sudo rm -r <edgeAgent storage folder>
sudo rm -r <edgeHub storage folder>
```

## 5 - Remove Local Certificates and Keys

```bash
sudo rm /var/lib/aziot/certd/certs/*
sudo rm /var/lib/aziot/keyd/keys/*
```

This forces IoT Edge to regenerate device and modules certificates. This is required especially when reprovisioning or fixing cert issues.

## 6 - Reprovision the device

Make sure that a valid IoT Edge configuration file is stored at `/etc/aziot/config.toml` so the device can reprovision.

```bash
sudo iotedge config apply
```

This will restart IoT Edge, reprovision the device and pull modules from the ACR if the images were deleted.

# Verify the Reset

Monitor the restart using these commands

```bash
# Check IoT Edge system logs
sudo iotedge system logs -- -f

# Check modules download and starting up
sudo iotedge system logs -- -f | grep image

# Check running modules (wait some time to let the modules download and starting up)
sudo iotedge list

# View the last 50 log lines of the edge agent
sudo iotedge logs edgeAgent --tail 50
```


## Full Reset Script

If you need to automate the process, here is a minimal shell script that combines all the steps above:

```bash
#!/bin/bash

echo "Resetting IoT Edge..."

echo "Stopping IoT Edge..."
iotedge system stop

echo "Deleting docker modules..."
docker rm -v -f $(docker ps -qa)

echo "Deleting edge Agent and Hub data..."
rm -r <edgeAgent storage folder> # Replace by correct folder
rm -r <edgehub storage folder> # Replace by correct folder

echo "Deleting IoT Edge certificates"
rm /var/lib/aziot/certd/certs/*
rm /var/lib/aziot/keyd/keys/*
```

Save it as `reset-iotedge.sh`, make it executable with `chmod +x reset-iotedge.sh`, and run it with `sudo ./reset-iotedge.sh`.


# Conclusion

Resetting an Azure IoT Edge device is a straightforward process: stop the runtime, remove all module containers, delete the modules storage directories, and optionally reprovision with a new configuration. Having this procedure scripted makes device lifecycle management significantly less stressful, whether you're running a handful of devices in the lab or thousands in the field.


