# Automatically Renaming a device Based on uts IP Address

When you deploy dozens (or hundreds) of devices in the field, naming consistency becomes critical.

In one of my projects, Azure IoT Edge devices were installed with static IP addresses following a predictable scheme: `192.168.X.Y`   

Where:

- The third octet [X] minus 2 represented the physical device number [N]
- The hostname needed to follow this convention: `device[N]`, so `device[X - 2]`

It was convenient for commissioning, as each device was able to directly configure itself with the correct hostname since it is plugged on the correct ethernet cable 

To do that, I wrote a simple Bash script that:
- Extracts the device IP address
- Computes the correct hostname
- Updates the system hostname if needed
- Resets Azure IoT Edge containers
- Reboots the device cleanly

Let’s break it down.

# 🎯 The Goal

Automatically ensure that:

- IP: 192.168.5.10
- Device number: 5 - 2 = 3
- Hostname: device3

# Procedure 

## 1 - Extract the Device IP Address

```bash
ip_address=$(ip route get 1 | sed -n 's/^.*src \([0-9.]*\) .*$/\1/p')
```

This command uses ip route get 1 to determine which interface is used for outbound traffic. It extracts the src IP address and returns something like `192.168.5.23`



## 2 - Extract the Third Octet

```bash
third_digit=$(echo $ip_address | cut -d. -f3)
```

This splits the IP on dots and retrieves field 3.

Example:

```bash
192.168.5.23
        ↑
        5
```

## 3 - Validate It’s a device Network

```bash
if [ "$third_digit" -gt 2 ]; then
```

Valid device IPs have a third octet greater than 2 (ower values belong to other devices)

## 4 - Compute the Correct Hostname

```bash
new_digit=$((third_digit - 2))
new_hostname="device$new_digit"
```


| Third Octet | Hostname |
| --- | --- |
| 3 | device1 |
| 4 | device2 |
| 5 | device3 |


## 5 - Change the Hostname 

Apply the new hostname :

```bash
hostnamectl set-hostname $new_hostname
```


## 6 - Reset Azure IoT Edge Containers

As we have seen in another article, an hostname change on an IoT Edge device require to reset the containers and recreate them

```bash
iotedge system stop && \
docker rm -f $(docker ps -aq -f "label=net.azure-devices.edge.owner=Microsoft.Azure.Devices.Edge.Agent") && \
iotedge config apply
```

# Script

Create the script file : `config_hostname_from_ip.sh`

```bash
#!/bin/bash

# Extract IP address
ip_address=$(ip route get 1 | sed -n 's/^.*src \([0-9.]*\) .*$/\1/p')

# Extract 3rd digit
third_digit=$(echo $ip_address | cut -d. -f3)

# Check 3rd digit is greater tha 2
if [ "$third_digit" -gt 2 ]; then

    # Get device number
    device_number=$((third_digit - 2))
    
    # Get new hostname
    new_hostname="device$device_number"
    
    echo "Hostname from IP ($ip_address) : $new_hostname"
    
    # get current hostname
    current_hostname=$(hostname)
    echo "Old hostname : $current_hostname"
    
    # Change the hostname only if it has changed
    if [ "$new_hostname" != "$current_hostname" ]; then
    
        echo "The hostname is wrong, change it to $new_hostname"
        hostnamectl set-hostname $new_hostname

        echo "Stopping IoT Edge..."
        iotedge system stop 

        echo "Removing IoT Edge containers..."
        docker rm -f $(docker ps -aq -f "label=net.azure-devices.edge.owner=Microsoft.Azure.Devices.Edge.Agent")
        echo ""

        echo "Restarting IoT Edge..."
        iotedge config apply
        echo ""
        
        echo "Reboot device..."
        reboot now
    else
        echo "Hostname correctly defined : $current_hostname"
    fi

else
	echo "This is not a device IP address"
fi
```

Make it executable 

```bash
chmod +x config_hostname_from_ip.sh
```

Use it 

```bash
./config_hostname_from_ip.sh 
```