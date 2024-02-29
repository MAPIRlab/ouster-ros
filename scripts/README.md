# Setup and Networking

Ouster gives you some tools to set up a direct connection to the sensor from you computer. I'd argue these are a bit obtuse and they should really provide a set of scripts to set this up automatically as a daemon. However, since I am also using this as a development tool, I don't want this always running in the background on my machines so I provide the directions below to setup the network connection.

### One time setup with IPv4

These are bash commands in Linux to setup the connection. These steps only need to happen the first time you set up the laser. After the first time, when you establish the network connection to the sensor, you can just select this created network profile. **Ensure the sensor is powered off and disconnected at this point.**

The `[eth name]` is the nework interface you're connecting to. On older Linux systems, that's `eth0` or similar. In newer versions, its `enp...` or `enx...` when you look at the output of `ifconfig`.

```bash
 ip addr flush dev [eth name]
 ip addr show dev [eth name]
```

The output you see from `show` should look something like `[eth name] ... state DOWN ...`. Its only important that you see `DOWN` and not `UP`. Next, lets setup a static IP address for your machine so you can rely on this in the future. Ouster uses the 10.5.5.* range, and I don't see a compelling reason to argue with it.

```bash
sudo ip addr add 10.5.5.1/24 dev [eth name]
```

Now, lets setup the connection. At this point you may now plug in and power on your sensor.

```bash
sudo ip link set [eth name] up
sudo addr show dev [eth name]
```

The output you see from `show` should look something like `[eth name] ... state UP ...`. Its only important that you see `UP` now and not `DOWN`. At this point, you've setup the networking needed for the one time setup.

### Connection with IPv4

We can setup the network connection to the sensor now with the proper settings. Note: This command could take up to 30 seconds to setup, be patient. If after a minute you see no results, then you probably have an issue. Start the instructions above over. Lets set up the network. There is a script file in this folder to launch this command easily:

```bash
sudo dnsmasq -C /dev/null -kd -F 10.5.5.50,10.5.5.100 -i [eth name] --bind-dynamic
```

Instantly you should see something similar to:

```bash
dnsmasq: started, version 2.75 cachesize 150
dnsmasq: compile time options: IPv6 GNU-getopt DBus i18n IDN DHCP DHCPv6 no-Lua TFTP conntrack ipset auth DNSSEC loop-detect inotify
dnsmasq-dhcp: DHCP, IP range 10.5.5.50 -- 10.5.5.100, lease time 1h
dnsmasq-dhcp: DHCP, sockets bound exclusively to interface enxa0cec8c012f8
dnsmasq: reading /etc/resolv.conf
dnsmasq: using nameserver 127.0.1.1#53
dnsmasq: read /etc/hosts - 10 addresses
```

You need to wait until you see something like:

```bash
dnsmasq-dhcp: DHCPDISCOVER(enxa0cec8c012f8) [HWaddr]
dnsmasq-dhcp: DHCPOFFER(enxa0cec8c012f8) 10.5.5.87 [HWaddr]
dnsmasq-dhcp: DHCPREQUEST(enxa0cec8c012f8) 10.5.5.87 [HWaddr]
dnsmasq-dhcp: DHCPACK(enxa0cec8c012f8) 10.5.5.87 [HWaddr] os1-SerialNumXX
```

Now you're ready for business. Lets see what IP address it's on (10.5.5.87). Lets ping it

```bash
ping 10.5.5.87
```

and we're good to go!

### Using IPv6 with link local

Instead of having to configure `dnsmasq` and static addresses in the previous section, you can use link local IPv6 addresses.

1. Find the link local address of the Ouster. With avahi-browse, we can find the address of the ouster by browsing all non-local services and resolving them.
```bash
$ avahi-browse -arlt
+   eth2 IPv6 Ouster Sensor 992109000xxx                    _roger._tcp          local
+   eth2 IPv4 Ouster Sensor 992109000xxx                    _roger._tcp          local
=   eth2 IPv6 Ouster Sensor 992109000xxx                    _roger._tcp          local
   hostname = [os-992109000xxx.local]
   address = [fe80::be0f:a7ff:fe00:2861]
   port = [7501]
   txt = ["fw=ousteros-image-prod-aries-v2.0.0+20201124065024" "sn=992109000xxx" "pn=840-102144-D"]
=   eth2 IPv4 Ouster Sensor 992109000xxx                    _roger._tcp          local
   hostname = [os-992109000xxx.local]
   address = [192.168.90.2]
   port = [7501]
   txt = ["fw=ousteros-image-prod-aries-v2.0.0+20201124065024" "sn=992109000xxx" "pn=840-102144-D"]

```
As shown above, on interface `eth2`, the ouster has an IPv6 link local address of `fe80::be0f:a7ff:fe00:2861`.

To use link local addressing with IPv6, the standard way to add a scope ID is appended with a `%` character like so in `sensor.yaml`. Automatic detection for computer IP address can be used with an empty string.
```bash
lidar_ip: "fe80::be0f:a7ff:fe00:2861%eth2"
computer_ip: ""
```

Note that this feature is only available with the default driver version, configured by `driver_config.yaml`. When running the Tins-based driver (see the following sections), both the LiDAR and computer IP address must be specified in `tins_driver_config.yaml`.

### Usage with the default driver

Now that we have a connection over the network, lets view some data. After building your colcon workspace with this package, source the install space, then run:

```bash
ros2 launch ros2_ouster driver_launch.py
```

Make sure to update your parameters file if you don't use the default IPs (10.5.5.1, 10.5.5.87). You may also use the `.local` version of your ouster lidar. To find your IPs, see the `dnsmasq` output or check with `nmap -SP 10.5.5.*/24`.
An alternative tool is [avahi-browse](https://linux.die.net/man/1/avahi-browse): 

```bash
avahi-browse -arlt
```

Now that your connection is up, you can view this information in RViz. Open an RViz session and subscribe to the points, images, and IMU topics in the laser frame. When trying to visualize the point clouds, be sure to change the Fixed Frame under Global Options to "laser_data_frame" as this is the default parent frame of the point cloud headers.

When the driver configures itself, it will automatically read the metadata parameters from the Ouster. If you wish to save these parameters and use them with captured data (see the next section) then you can save the data to a specific location using the `getMetadata` service that the driver offers. To use it, run the driver with a real Ouster LiDAR and then make the following service call:

```bash
ros2 service call /ouster_driver/get_metadata ouster_msgs/srv/GetMetadata "{metadata_filepath: "/path/to/your/metadata.json"}"
```

The driver will then save all the required metadata to the specified file and dump the same metadata as a JSON string to the terminal. Alternatively, the service can be called without specifying a filepath (see below) in which case no file will be saved, and the metadata will still be printed to terminal. Copying this string and manually saving it to a .json file is also a valid way to generate a metadata file.  

```bash
ros2 service call /ouster_driver/get_metadata ouster_msgs/srv/GetMetadata
```

Have fun!