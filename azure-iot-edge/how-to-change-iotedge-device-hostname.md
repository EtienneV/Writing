# How to Change the Hostname of an Azure IoT Edge Device

When deploying fleets of industrial devices, naming conventions matter.

Whether you're managing local controllers, smart kiosks, or industrial gateways, you’ll eventually need to rename a device to match your production logic, for example:

```
gateway1  
screen-2-controller  
metering-room
```

If your device is running Azure IoT Edge, changing the hostname is not just cosmetic. The hostname is used internally by the runtime, certificates, and container networking. Doing it incorrectly can break your deployment.

# Step-by-Step Procedure
This procedure is for Linux-based devices

## 1 - Change the hostname

Change the hostname using hostnamectl

```bash
hostnamectl set-hostname <new-hostname>
```

You can check the new hostname with `hostnamectl`

At this point, the system hostname is updated, but IoT Edge is still running with the old configuration.

## 2 - Cleanly restart IoT Edge

We need to:

- Stop IoT Edge
- Remove Edge-managed containers
- Re-apply configuration

This way, all IoT Edge containers are recreated using the new hostname configuration

```bash
iotedge system stop && \
docker rm -f $(docker ps -aq -f "label=net.azure-devices.edge.owner=Microsoft.Azure.Devices.Edge.Agent") && \
iotedge config apply
```

# Automating with a script

Instead of doing this manually every time, let’s create a reusable script.

In this example, the script takes a device number and sets the hostname to `device-<device number>` (for example, `device-2`)

Create the script file : `config_hostname.sh`

```bash
#!/bin/bash

while getopts n: flag
do
    case "${flag}" in
        n) DEVICE_NUMBER=${OPTARG};;
    esac
done

echo "Configuration of the device number ${DEVICE_NUMBER}"
echo ""

if [[ $DEVICE_NUMBER != "" ]]; then

  echo "Set hostname to device-${DEVICE_NUMBER} ..."
  hostnamectl set-hostname "device-${DEVICE_NUMBER}"
  echo ""

  echo "Stopping IoT Edge..."
  iotedge system stop 
  echo ""

  echo "Removing IoT Edge containers..."
  docker rm -f $(docker ps -aq -f "label=net.azure-devices.edge.owner=Microsoft.Azure.Devices.Edge.Agent") 
  echo ""

  echo "Restarting IoT Edge..."
  iotedge config apply
  echo ""
fi
```

Make it executable 

```bash
chmod +x config_hostname.sh
```

Use it 

```bash
./config_hostname.sh -n 2
```


