# Container networking

The idea of this runbook is to explore the use of Linux namespaces and it's role on creating containers. 

## Setup & Creating a namespaces
```bash
CONTAINER_NAME="mycontainer"
CONTAINER_DIR="./${CONTAINER_NAME}"


## Create container directory
mkdir -p "${CONTAINER_DIR}"

## Use debootstrap to create a minimal Debian root filesystem
sudo apt update && sudo apt install debootstrap -y
sudo debootstrap --arch amd64 bookworm "${CONTAINER_DIR}" http://deb.debian.org/debian

## Create network namespace
sudo ip netns add ${CONTAINER_NAME}

## Create container namespace (it will isolate resources from host machine)
sudo unshare --fork --pid --net=/var/run/netns/mycontainer --mount --uts --cgroup --ipc --root=${CONTAINER_DIR} bash
```

# Setup container
```bash
## Run mount, set hostname and assign IP address
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev
mount -t devpts devpts /dev/pts -o gid=5,mode=620
hostname mycontainer
#ip addr add 172.17.0.100/24 dev eth0 # Configure container IP
#ip route add default via 172.17.0.1 # Set default gateway
#exec /bin/bash -l

## HOST - Get all namespaces and related  PID 
sudo lsns

## HOST - Create a bridge interface (if it doesn't exist)
sudo ip link add br0 type bridge
sudo ip addr add 172.17.0.1/24 dev br0
sudo ip link set dev br0 up

## HOST - create veth pair and assign it to the 
sudo ip link add veth1 type veth peer name veth2 netns ${CONTAINER_NAME}
sudo ip link set dev veth1 up
sudo ip link set dev veth1 master br0

## CHILD
ip addr add 172.17.0.100/24 dev veth2 # Configure container IP
ip link set dev veth2 up
ip link set lo up
ip route add default via 172.17.0.1 # Set default gateway
```

# Test child networking (inbound)
```bash
# CONTAINER
# Insid the container, run a ping command
ping 8.8.8.8

# Hmm, not working, no internet connection...

# HOST
# Let's install tshark and see what the network packages going out looks like
sudo apt install tshark -y

# Let's list all interfaces
sudo tshark -D

# Lets get some packages from our bridge interface
sudo tshark -i br0

# Great, we are getting packages:
#Capturing on 'br0'
# ** (tshark:30699) 23:20:56.697008 [Main MESSAGE] -- Capture started.
# ** (tshark:30699) 23:20:56.697113 [Main MESSAGE] -- File: "/tmp/wireshark_br0TQR8Z2.pcapng"
#    1 0.000000000 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1036/3076, ttl=64
#    2 1.023682065 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1037/3332, ttl=64
#    3 2.046162457 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1038/3588, ttl=64
#    4 3.069509104 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1039/3844, ttl=64

# Lets have a look at the host main interface
sudo tshark -i enp0s3 host 8.8.8.8

# Hmm...no packages
#Capturing on 'enp0s3'
# ** (tshark:30724) 23:21:51.765441 [Main MESSAGE] -- Capture started.
# ** (tshark:30724) 23:21:51.765561 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s3HWD7Z2.pcapng"

# We might need to allow the host to forward packages that are not from itself (10.0.2.15) 
# Enable IP forwarding
sudo sysctl net.ipv4.ip_forward=1 

# Lets try again...
#Capturing on 'enp0s3'
# ** (tshark:30754) 23:23:24.917198 [Main MESSAGE] -- Capture started.
# ** (tshark:30754) 23:23:24.917283 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s3ARGA02.pcapng"
#    1 0.000000000 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1178/39428, ttl=63
#    2 1.022383647 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1179/39684, ttl=63
#    3 2.047483609 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1180/39940, ttl=63
#    4 3.073494346 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1181/40196, ttl=63
#    5 4.097752694 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1182/40452, ttl=63
#    6 5.121516792 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=1183/40708, ttl=63

# We are getting the requests out, but we are not getting the responses.
# Thats is because the router won't be able to identify who is 172.17.0.100 on the network
# This IP address is particular to our container, that lives inside 10.0.2.15.
# We will need to workaround this by using another Linux functionality: iptables
# It implements package filter rules to filter and manipulate network packages

# Update iptables to forward outbound traffic from container
sudo iptables --table nat -A POSTROUTING -s 172.17.0.0/24 ! -o br0 -j MASQUERADE

# Lets test again
sudo tshark -i enp0s3 host 8.8.8.8

#Capturing on 'enp0s3'
# ** (tshark:31141) 00:52:02.493177 [Main MESSAGE] -- Capture started.
# ** (tshark:31141) 00:52:02.493256 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s3Q65N02.pcapng"
#    1 0.000000000    10.0.2.15 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6280/34840, ttl=63
#    2 0.026917679      8.8.8.8 → 10.0.2.15    ICMP 98 Echo (ping) reply    id=0x0019, seq=6280/34840, ttl=255 (request in 1)
#    3 1.004672037    10.0.2.15 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6281/35096, ttl=63
#    4 1.033187807      8.8.8.8 → 10.0.2.15    ICMP 98 Echo (ping) reply    id=0x0019, seq=6281/35096, ttl=255 (request in 3)
#    5 2.009580967    10.0.2.15 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6282/35352, ttl=63
#    6 2.037015186      8.8.8.8 → 10.0.2.15    ICMP 98 Echo (ping) reply    id=0x0019, seq=6282/35352, ttl=255 (request in 5)
#    7 3.016842995    10.0.2.15 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6283/35608, ttl=63
#    8 3.046764418      8.8.8.8 → 10.0.2.15    ICMP 98 Echo (ping) reply    id=0x0019, seq=6283/35608, ttl=255 (request in 7)
#    9 4.023203294    10.0.2.15 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6284/35864, ttl=63

# If we capture on the bridge device, we can see the container IP address
sudo tshark -i br0 host 8.8.8.8

#Capturing on 'br0'
# ** (tshark:31168) 00:55:11.721410 [Main MESSAGE] -- Capture started.
# ** (tshark:31168) 00:55:11.721497 [Main MESSAGE] -- File: "/tmp/wireshark_br09D07Z2.pcapng"
#    1 0.000000000 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6453/13593, ttl=64
#    2 0.028362719      8.8.8.8 → 172.17.0.100 ICMP 98 Echo (ping) reply    id=0x0019, seq=6453/13593, ttl=254 (request in 1)
#    3 1.001651531 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6454/13849, ttl=64
#    4 1.029901240      8.8.8.8 → 172.17.0.100 ICMP 98 Echo (ping) reply    id=0x0019, seq=6454/13849, ttl=254 (request in 3)
#    5 2.005381437 172.17.0.100 → 8.8.8.8      ICMP 98 Echo (ping) request  id=0x0019, seq=6455/14105, ttl=64
#    6 2.033813567      8.8.8.8 → 172.17.0.100 ICMP 98 Echo (ping) reply    id=0x0019, seq=6455/14105, ttl=254 (request in 5)
```

# Container running a HTTP server (outbound)
```bash
# Set DNS, so we can resolve server names
cat > /etc/resolv.conf << EOF
nameserver 8.8.8.8
EOF

apt update && apt install nginx

/etc/init.d/nginx start


```