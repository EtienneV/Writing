# How to Transfer a Docker Image via USB

Sometimes, devices cannot pull images directly from a container registry:
- No internet access
- Restricted firewall rules
- Limited bandwidth
- Industrial or isolated environments

In those cases, you can manually export a Docker image from a connected machine and import it onto the target device.   
Here we show how to get a docker image from a device to install it to another one using an USB key.

# Step 1 : Export the docker image

First, you need to get the docker image from a device

## Mount the USB key

```bash
mkdir /media/usb
mount /dev/sda1 /media/usb
```

## Export the Image

This command permits to export a docker image to a tar file : 

```bash
docker save -o <FILE_PATH> <IMAGE>
```

To get the `<IMAGE>`, look at the docker images list with `docker images` : 

```bash
REPOSITORY                 TAG          IMAGE ID       SIZE
registry.example.com/app   1.0.0-arm64  abc123456789   850MB
registry.example.com/db    2.1.3        def987654321   320MB
```

The full image reference format is: `<REPOSITORY>:<TAG>`

Example : 

```bash
docker save -o /media/usb/app_1.0.0-arm64.tar registry.example.com/app:1.0.0-arm64
```

Once complete, safely unmount the USB key.

# Step 2 : Import the Image on the Target Device

Move the USB key to the offline or restricted device.

## Mount the USB Key

```bash
mkdir /media/usb
mount /dev/sda1 /media/usb
```

## Load the Image

This command permits to load a docker image from a tar file :

```bash
docker load -i <FILE>
```

Example : 

```bash
docker load -i /media/usb/app_1.0.0-arm64.tar
```

## Check if image was correctly loaded

You should see the image in the list with : `docker images`

