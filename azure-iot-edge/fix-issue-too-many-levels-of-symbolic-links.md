# Fixing Docker “Too Many Levels of Symbolic Links” on IoT Edge Devices

If you’re running Azure IoT Edge on a Linux device and suddenly your module won’t start with the following error:

```bash
failed to register layer: error creating overlay mount to /var/lib/docker/overlay2/.../merged: too many levels of symbolic links
```

You’re likely dealing with a corrupted Docker overlay filesystem.

This typically happens after a sudden power cut, an interrupted docker pull or a forced reboot during image extraction. In that case, you may end up with inconsistent overlay mount, or corrupted Docker metadata. Once that happens, Docker cannot properly mount the image layer anymore, and every restart attempt will fail.

In IoT Edge environments, this completely blocks module deployment.

To fix that, you can reset the Docker storage directory. Be careful as it removes all Docker images and containers stored locally.

# 1 - Stop IoT Edge 

```bash
iotedge system stop
```

# 2 - Backup the Docker Directory

```bash
cd /var/lib
mv docker docker.old
```

Rename the folder instead of deleting it directly, so it allows rollback if necessary.

# 3 - Recreate a Clean Docker Directory

```bash
mkdir docker
chmod 710 docker
```

Recreate the folder with proper permissions.

# 4 - Reboot the Device

```bash
reboot now
```

# 5 - Reapply IoT Edge Configuration

```bash
iotedge config apply
```

IoT Edge will pull fresh module images, and recreate containers from scratch

# 6 - If Everything Works

Once modules are working again, you can remove the old corrupted storage

```bash
cd /var/lib
rm -r docker.old
```




