# Container networking

The idea of this runbook is to explore the use of Linux namespaces and it's role on creating containers. 

## Setup & Creating a namespaces
```bash
## Create container directory
mkdir -p "./mycontainer"

## Use debootstrap to create a minimal Debian root filesystem
sudo apt update && sudo apt install debootstrap -y
sudo debootstrap --arch amd64 bookworm "./mycontainer" http://deb.debian.org/debian

## Lets get a list of all network namespaces at the moment
## For now, we will focus on network namespaces
sudo lsns -t net

## Create network namespace
sudo ip netns add mycontainer

## Create container namespace (it will isolate resources from host machine)
sudo unshare --fork --pid --net=/var/run/netns/mycontainer --mount --uts --cgroup --ipc --root=./mycontainer bash
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
sudo lsns -t net

## HOST - Create a bridge interface (if it doesn't exist)
sudo ip link add br0 type bridge
sudo ip addr add 172.17.0.1/24 dev br0
sudo ip link set dev br0 up

## HOST - create veth pair and assign it to the 
sudo ip link add veth1 type veth peer name eth0 netns mycontainer
sudo ip link set dev veth1 up
sudo ip link set dev veth1 master br0

## CHILD
ip addr add 172.17.0.100/24 dev eth0 # Configure container IP
ip link set dev eth0 up
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
sudo tshark -i eth0 host 8.8.8.8

# Hmm...no packages
#Capturing on 'eth0'
# ** (tshark:30724) 23:21:51.765441 [Main MESSAGE] -- Capture started.
# ** (tshark:30724) 23:21:51.765561 [Main MESSAGE] -- File: "/tmp/wireshark_eth0HWD7Z2.pcapng"

# We might need to allow the host to forward packages that are not from itself (10.0.2.15) 
# Enable IP forwarding
sudo sysctl net.ipv4.ip_forward=1 

# Lets try again...
#Capturing on 'eth0'
# ** (tshark:30754) 23:23:24.917198 [Main MESSAGE] -- Capture started.
# ** (tshark:30754) 23:23:24.917283 [Main MESSAGE] -- File: "/tmp/wireshark_eth0ARGA02.pcapng"
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
sudo tshark -i eth0 host 8.8.8.8

#Capturing on 'eth0'
# ** (tshark:31141) 00:52:02.493177 [Main MESSAGE] -- Capture started.
# ** (tshark:31141) 00:52:02.493256 [Main MESSAGE] -- File: "/tmp/wireshark_eth0Q65N02.pcapng"
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

# Install nginx 
apt update && apt install nginx -y

# Start nginx service manually
/etc/init.d/nginx start


# HOST
# Lets see what is going on on port 8080 of the host if we make a request from ouside the host
sudo tshark -i eth0 port 8080

# Output:
#Running as user "root" and group "root". This could be dangerous.
#Capturing on 'eth0'
#    1 0.000000000  172.21.16.1 → 172.21.28.117 TCP 66 53783 → 8080 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 WS=256 SACK_PERM
#    2 0.000029630 172.21.28.117 → 172.21.16.1  TCP 54 8080 → 53783 [RST, ACK] Seq=1 Ack=1 Win=0 Len=0
#    3 0.002759672  172.21.16.1 → 172.21.28.117 TCP 66 53784 → 8080 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 WS=256 SACK_PERM
#    4 0.002776793 172.21.28.117 → 172.21.16.1  TCP 54 8080 → 53784 [RST, ACK] Seq=1 Ack=1 Win=0 Len=0
# ...

# Configure iptables to route traffic from port 8080 to container port 80 (port mapping)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.100:80

# Lets see what happens after 
sudo tshark -i eth0 port 8080

# Output:
#Running as user "root" and group "root". This could be dangerous.
#Capturing on 'eth0'
#    1 0.000000000  172.21.16.1 → 172.21.28.117 TCP 66 54034 → 8080 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 WS=256 SACK_PERM
#    2 0.000063190 172.21.28.117 → 172.21.16.1  TCP 66 8080 → 54034 [SYN, ACK] Seq=0 Ack=1 Win=64240 Len=0 MSS=1460 SACK_PERM WS=128
#    3 0.000177251  172.21.16.1 → 172.21.28.117 TCP 66 54035 → 8080 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 WS=256 SACK_PERM
#    4 0.000194826  172.21.16.1 → 172.21.28.117 TCP 54 54034 → 8080 [ACK] Seq=1 Ack=1 Win=65280 Len=0
#    5 0.000200801 172.21.28.117 → 172.21.16.1  TCP 66 8080 → 54035 [SYN, ACK] Seq=0 Ack=1 Win=64240 Len=0 MSS=1460 SACK_PERM WS=128
#    6 0.000258881  172.21.16.1 → 172.21.28.117 HTTP 532 GET / HTTP/1.1
#    7 0.000274121 172.21.28.117 → 172.21.16.1  TCP 54 8080 → 54034 [ACK] Seq=1 Ack=479 Win=64128 Len=0
#    8 0.000302512  172.21.16.1 → 172.21.28.117 TCP 54 54035 → 8080 [ACK] Seq=1 Ack=1 Win=65280 Len=0
#    9 0.000590174 172.21.28.117 → 172.21.16.1  HTTP 712 HTTP/1.1 200 OK  (text/html)
#   10 0.026102167  172.21.16.1 → 172.21.28.117 HTTP 464 GET /favicon.ico HTTP/1.1
#   11 0.026266877 172.21.28.117 → 172.21.16.1  HTTP 427 HTTP/1.1 404 Not Found  (text/html)
#   12 0.076813086  172.21.16.1 → 172.21.28.117 TCP 54 54034 → 8080 [ACK] Seq=889 Ack=1032 Win=64256 Len=0
```