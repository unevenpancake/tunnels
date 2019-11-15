# tunnels

This is my presentation about Tunnels.

1. SSH
ssh-keygen -t ed25519

2. Wireguard
#Open up security groups
sudo yum update -y
sudo reboot # unless there were no updates
sudo curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
sudo yum install epel-release
sudo yum install wireguard-dkms wireguard-tools
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.1/24
#ip address add dev wg0 192.168.2.1 peer 192.168.2.2
#wg setconf wg0 myconfig.conf
#Server
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.1/24
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
cat publickey
sudo wg set wg0 listen-port 51820 private-key ./privatekey peer pp1Fu4U6LzVGUgkAlu9CNBONJeP7G/7Forx/zoVvYS0= allowed-ips 192.168.7.2/32
sudo wg showconf wg0
sudo ip link set up dev wg0
#Client
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.2/24
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
cat publickey
sudo wg set wg0 private-key ./privatekey peer LYhSr2Q3mQcEVijh/nHZ44zk8SWQIqnYM7X8ikyw+D0= allowed-ips 192.168.7.0/24 endpoint 3.82.199.162:51820 persistent-keepalive 30
sudo wg showconf wg0
sudo ip link set up dev wg0
#wg show
