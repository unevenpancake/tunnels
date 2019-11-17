# Tunnels

How to use SSH, OpenVPN, and Wireguard.  Please note that wherever IP's are used through this you should use your own (especially if they are public IP's, this was built using AWS boxes that I randomly acquired public IP's for.)

## SSH
### Generate a secure keypair
Please use a password.  There is a time for not using a password, but ssh-agent exists to cache an unlocked private key while you are using it so you can conveniently save it
```
ssh-keygen -t ed25519
```
### Copy your newly generated key to a target server
```
ssh-copy-id centos@54.165.252.132
```
```
ssh -L 8080:localhost:9090 centos@54.165.252.134#Go out to a box and connect to a "hidden" service
ssh -R 2222:localhost:22 #Reverse shell out
ssh -D 1337 #Make all of your traffic behave as though it were coming from a different box (SOCKS5)
```

## OpenVPN
### Install the Packages
```
sudo yum install -y openvpn easy-rsa vim
```
### Setup PKI
```
sudo cp -r /usr/share/easy-rsa /etc/openvpn
cd /etc/openvpn/easy-rsa/3
```
```
sudo vim vars
```
```
set_var EASYRSA                 "$PWD"
set_var EASYRSA_PKI             "$EASYRSA/pki"
set_var EASYRSA_DN              "cn_only"
set_var EASYRSA_REQ_COUNTRY     "ID"
set_var EASYRSA_REQ_PROVINCE    "Jakarta"
set_var EASYRSA_REQ_CITY        "Jakarta"
set_var EASYRSA_REQ_ORG         "hakase-labs CERTIFICATE AUTHORITY"
set_var EASYRSA_REQ_EMAIL       "openvpn@hakase-labs.io"
set_var EASYRSA_REQ_OU          "HAKASE-LABS EASY CA"
set_var EASYRSA_KEY_SIZE        2048
set_var EASYRSA_ALGO            rsa
set_var EASYRSA_CA_EXPIRE       7500
set_var EASYRSA_CERT_EXPIRE     365
set_var EASYRSA_NS_SUPPORT      "no"
set_var EASYRSA_NS_COMMENT      "HAKASE-LABS CERTIFICATE AUTHORITY"
set_var EASYRSA_EXT_DIR         "$EASYRSA/x509-types"
set_var EASYRSA_SSL_CONF        "$EASYRSA/openssl-1.0.cnf"
set_var EASYRSA_DIGEST          "sha256"
```
*You will need a password you remember on this step; you need to enter it everytime you create new certificate*
```
sudo chmod +x vars
sudo ./easyrsa init-pki
```
```
sudo ./easyrsa build-ca
```
### Build a server certificate
```
sudo ./easyrsa gen-req openvpnserver nopass
sudo ./easyrsa sign-req server openvpnserver
```
### Validate the server certificate
*You will need sudo because the CA is owned by root*
```
sudo openssl verify -CAfile pki/ca.crt pki/issued/openvpnserver.crt
```
### Build a client certificate
```
sudo ./easyrsa gen-req openvpnclient nopass
sudo ./easyrsa sign-req client openvpnclient
```
### Verify the client
```
sudo openssl verify -CAfile pki/ca.crt pki/issued/openvpnclient.crt
```
### Generate a Diffie-Hellman Group
```
sudo ./easyrsa gen-dh
```

### Copy the server keys to the OpenVPN directory
You can generate certificates and private keys in this directory to start with, but most guides will have you store them in one location as a archive and copy them where needed.  This also enables you to generate this information on one system and copy it to another if you wanted to secure it in a secrets management tool such as Hashicorp Vault or AWS Secrets Manager.
```
sudo cp pki/ca.crt /etc/openvpn/server/
sudo cp pki/issued/openvpnserver.crt /etc/openvpn/server/
sudo cp pki/issued/openvpnserver.crt /etc/openvpn/server/
sudo cp pki/private/openvpnserver.key /etc/openvpn/server/
sudo cp pki/dh.pem /etc/openvpn/server/
```

### Generate a server configuration
```
sudo vim /etc/openvpn/server.conf
```
```
#OpenVPN Port, Protocol and the Tun
port 1194
proto udp
dev tun

# OpenVPN Server Certificate - CA, server key and certificate
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/openvpnserver.crt
key /etc/openvpn/server/openvpnserver.key

#DH and CRL key
dh /etc/openvpn/server/dh.pem
# Only use when we introduce a CRL
#crl-verify /etc/openvpn/server/crl.pem

# Network Configuration - Internal network
# Redirect all Connection through OpenVPN Server
server 10.10.1.0 255.255.255.0
push "redirect-gateway def1"

# Using the DNS from https://dns.watch
push "dhcp-option DNS 1.1.1.1"

#Enable multiple client to connect with same Certificate key
duplicate-cn

# TLS Security
cipher AES-256-CBC
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA256:TLS-DHE-RSA-WITH-AES-128-GCM-SHA256:TLS-DHE-RSA-WITH-AES-128-CBC-SHA256
auth SHA512
auth-nocache

# Other Configuration
keepalive 20 60
persist-key
persist-tun
comp-lzo yes
daemon
user nobody
group nobody

# OpenVPN Log
log-append /var/log/openvpn.log
verb 3
```
### Allow the server to route (you will need to change a kernel setting via sysctl)
```
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```
### Allow OpenVPN on the firewall; allow all tunnel traffic to pass
```
firewall-cmd --permanent --zone=trusted --add-interface=tun0
```
### Allow server to act as a gateway to other networks
Remember that OpenVPN is well-known on UDP 1194.  If you want to change your port, you will need to specify ports by number and protocol in the firewall.
```
firewall-cmd --permanent --add-service=openvpn --zone=external
firewall-cmd --permanent --change-interface=eth0 --zone=external
```
### Make sure you check the cloud firewall as well
For example, in AWS there is security groups that also will need to allow UDP 1194 in order to work.


## Reload firewall
firewall-cmd --reload

### Start and enable server all the time
```
sudo systemctl enable --now openvpn@server #(this is defined within the systemd unit file) (https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Description)
```
### Check to see if it's listening on UDP 1194
```
sudo ss -ulnp (UDP, listening, no port service names (use numbers), show process using port)
```
### Build a client OpenVPN file in a single file
You will need to copy the ca public certificate in the `<ca>` block, the client public certificate to the `<certificate>`, and the private key to the `<key>` block.  You can also build this file without adding this information which can make it easier to store in a git repo.  If you do this simply reference these files as the server config above references them.  This can also make certificate rotation much easier.  It can be very helpful to have all of this is one file because then you can simply provide a one file to anyone who wants to connect and run `sudo openvpn <filename>` to connect.
```
vim client.ovpn
```
```
client
dev tun
proto udp

remote 3.82.199.162 1194

cipher AES-256-CBC
auth SHA512
auth-nocache
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA256:TLS-DHE-RSA-WITH-AES-128-GCM-SHA256:TLS-DHE-RSA-WITH-AES-128-CBC-SHA256

resolv-retry infinite
compress lzo
nobind
persist-key
persist-tun
mute-replay-warnings
verb 3

#CACert
<ca>
-----BEGIN CERTIFICATE-----
MIIDNTCCAh2gAwIBAgIJAPKl45OssbghMA0GCSqGSIb3DQEBCwUAMBYxFDASBgNV
BAMMC0Vhc3ktUlNBIENBMB4XDTE5MTExNTE4MjI1N1oXDTQwMDUyODE4MjI1N1ow
FjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQC16cKr0wA7qmCbBax64d2gnffaH7m/WkavA0lq49NoGVKgx00AB4pT
XI0JqQGKQeHAprXVOUl7dBlVgKhW2MvD8SMXuugAcFrtjQD/k4Mt77chjG5Atcf9
+rgTl8y6Qd+m//mAnGrRZk5mphcpKinZIcrmO6g2fB7U3avhXzQWo8ci+QX19xyE
jHUtco/sGpSL3Ua6TatC3Gy0Yjt04+PykdSdOPGI3hA2HLfSs/pKCRbknGs36aSw
O1U7uQxIi7AtP46Nfz9KeB/aMppkajlAYyA2YNeCk7+ExG95IIrMxTXFinCOKiZY
vrPNu0K9vWLvzBcJ3r+GWgC4COhpnS6tAgMBAAGjgYUwgYIwHQYDVR0OBBYEFBts
RQoNX2EnFHJp1eU6LSHXHlaCMEYGA1UdIwQ/MD2AFBtsRQoNX2EnFHJp1eU6LSHX
HlaCoRqkGDAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQYIJAPKl45OssbghMAwGA1Ud
EwQFMAMBAf8wCwYDVR0PBAQDAgEGMA0GCSqGSIb3DQEBCwUAA4IBAQAjZDwrV84x
CzM46NEZGoYl4JdhfTOyu7N6bIKCouaUhG6+bvx/RW6/zEqaCjGGWuAh9/U+RPiw
xbx56hKnZrS24kaJ1OxWUwSQtuZsslxFa8nD1cos81Jh60Ik+/+wsPCFi/HEVhXO
ymwqaeFlv+AKoy9czTwBUOif6A5hItJgoW7pYE2ByFQ2x9hyavmL4iruPhct3iD4
AkVrOBERC2hbLgMkgnIbH52oZCjv8jLMQbDzV7KikCIqfKg1f/0otj4JtiaP706t
wUCNt4erA6YXqoWHE8hPjYcgvX66dt76JhtHURyhEHEEGU5NYN75q2JQkXBdpWiR
AHrcnVUU3kKv
-----END CERTIFICATE-----
</ca>
#openvpnclient cert
<cert>
-----BEGIN CERTIFICATE-----
MIIDUDCCAjigAwIBAgIQS5QK+aQwH+z9CeqqjWW/ZzANBgkqhkiG9w0BAQsFADAW
MRQwEgYDVQQDDAtFYXN5LVJTQSBDQTAeFw0xOTExMTUxODI3MDBaFw0yMDExMTQx
ODI3MDBaMBgxFjAUBgNVBAMMDW9wZW52cG5jbGllbnQwggEiMA0GCSqGSIb3DQEB
AQUAA4IBDwAwggEKAoIBAQDVv2cPd5oqhAoT7iu77fbhVV7ayHyWPHtySl6jxb12
TM+BDYdNg2GgM1ET5ROSTI3utI5u+XEpBDACDQ7CBF4K4J/Hsm9nq06dVHM0xhgl
DOAyuhfHGYjlywN61mW04vlLZntIa99aqyzpzD3DzPSApRdSoE8cXgKf+MAH7jHx
xwTEhGYRleAJNWZx0wld3OFhE+d6/X+c2v7molg43Wf1aJ9K7h5mrZv8fFug1/X1
hiDVbMQS0yE/tA95enXxIDmQdZ+TecGjV1NTmQOiPc4DCXEu3QF8Bj+jEgMzySem
0/tHJABCuSZerfMUiBOkQhx4Pb9d7NJpOm+NPP2RnhbZAgMBAAGjgZcwgZQwCQYD
VR0TBAIwADAdBgNVHQ4EFgQU7LSaPqurpAlH6t0HKn1qCVlI1ZgwRgYDVR0jBD8w
PYAUG2xFCg1fYScUcmnV5TotIdceVoKhGqQYMBYxFDASBgNVBAMMC0Vhc3ktUlNB
IENBggkA8qXjk6yxuCEwEwYDVR0lBAwwCgYIKwYBBQUHAwIwCwYDVR0PBAQDAgeA
MA0GCSqGSIb3DQEBCwUAA4IBAQA3V5ptvyP3XVeg9lUFdTyyDT4yR0F2eQZHbJdt
FvUHDzxfWkPRd8a4kccwVI4YyRziiF0QeJLxAUo8nXXaYnhdUgj/HOldD3SOeyCV
7uRVnSstgq2pENsex7ClCdXNDCEV20JWPT/JpkfP+U+L/kMYO/GtE9zAQwcG/YZ9
bXIn4zRHSADtI8mQ3RNHdliYDxEUWpXB6sIJzoM09VkC07O76Rodaw8TaPP7d/6/
D9/QRgaRA+sTffB2zUcaZwj5mShf/SODbBmaQHqbfrQ9ewiYijzXDWRjNOjTJiQc
VW4R23R2pVUaeRCXqyuSU6fl669Z+Q2IACuv8KRtgRkJZWO+
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDVv2cPd5oqhAoT
7iu77fbhVV7ayHyWPHtySl6jxb12TM+BDYdNg2GgM1ET5ROSTI3utI5u+XEpBDAC
DQ7CBF4K4J/Hsm9nq06dVHM0xhglDOAyuhfHGYjlywN61mW04vlLZntIa99aqyzp
zD3DzPSApRdSoE8cXgKf+MAH7jHxxwTEhGYRleAJNWZx0wld3OFhE+d6/X+c2v7m
olg43Wf1aJ9K7h5mrZv8fFug1/X1hiDVbMQS0yE/tA95enXxIDmQdZ+TecGjV1NT
mQOiPc4DCXEu3QF8Bj+jEgMzySem0/tHJABCuSZerfMUiBOkQhx4Pb9d7NJpOm+N
PP2RnhbZAgMBAAECggEAXR5YgLWDNTh2x34AEYw2/K3beAbVuAG7aewaVNDFnG8U
C03gfxVYh5kzni4zG448WxzP3GrRMKRBYfNcVYvfiG+ZTD9hJ1HLGuF6mygdxq5Y
UeEekL+AE1QhPPeAMZCcOIv583ADSxW9qFExK0bz0cOaaIWsUVhnXlfZGNtdaM2W
TbF92jXuWIE78WsFxuHF8pRmKltfoDefbhCvA0BcOXZi6LxhGvaDBTmrFcI1WYp0
/ULdU/Z945U0i72ZsW4WNzu9NravIJOyoDwxU1Rb9KLHiByXH5RDasEmmwLiDpv+
ZzITJr2WtsjMgXEjXUxUbgjXiKNmcpc6BuiuBhWVgQKBgQDxI5h5qDF253ZiZile
A4V8FOiicZATECS0RjyHX/IJCSJoUEZBU+1/HEzpPrabs8bDX+p+CHdU9XJoMlpx
LZ+Eq9UNz6PtuMbbB7imajW4sWN/Bo64f92PRiDjZREjVcCjo3MLvkQy6un/yZLa
jFMQAu9RNgL4I1ZiRNXIt7XA0QKBgQDi66iQGuuOb+j+FgTvy1uqTMREDX5MiMYK
KS3mi/B+zNUi0lAxATcv14/8QYjlTRwU/G2C1ZMvUsqotoNzxIAAXpRsw/h+TXlx
PYy6uZJ34wFNnaXrsmpr9Uj7nhuGhcMA0OL5w7C3jaEBctf8PAtUxy0db5wISekj
px4z2V03iQKBgQCQM2AgCFOkLmBeEYfVX7e4bux7D/w/Wh0I7SOPNPIRMzQvOyn4
MQ9KPwtDRCyBSe2nsjkRK1DpLmo/IzVwjv7goL0kqDH4m9HW83QZmFQN4Y6FTM+W
R2igICjUswCfp80uTjUjJaG07UQHoWw/Y0Dcx1SDtQ/rgX5L/6v0ft+isQKBgDyY
y8m3tqGx1tlLTgQvHQpsN5kotUqA18nM11oSkqV504zZ9tovRep7uRKW+ZSqM86S
3jerCwP/KulE2/OlTL1MhHxLFOe9jqmj0xnmBmwHbcipSa6YVX0A4n126kjRHZLx
NTuXe3B43L8DSRQtgKUiDzUmIdfAzQZdUV5tNExpAoGAXFs06dykKDY5TxIdjlxp
sizj0L8KbwTMHIHXnOWYSyQgbxjwp37feNFTAc+ZHUqL9mAIGaChNgyROJi65YY+
e+BCYV14C1+J5kN6NJNtWm8+chiqkyO+oKEYf8mnTMCQ9EBf392aEOHpYw2CcDVw
X2ADsnWFJfhztbnzEoLSr70=
-----END PRIVATE KEY-----
</key>
```
### Connect!
```
sudo openvpn client.ovpn
```

## Wireguard
sudo yum update -y
sudo reboot # unless there were no updates
sudo curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
sudo yum install epel-release
sudo yum install wireguard-dkms wireguard-tools
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.1/24
#ip address add dev wg0 192.168.2.1 peer 192.168.2.2
#wg setconf wg0 myconfig.conf
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
cat publickey
####Establish Link manually
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.1/24
sudo wg set wg0 listen-port 51820 private-key ./privatekey peer pp1Fu4U6LzVGUgkAlu9CNBONJeP7G/7Forx/zoVvYS0= allowed-ips 192.168.7.2/32
sudo wg showconf wg0
sudo ip link set up dev wg0
#Client
sudo ip link add dev wg0 type wireguard
sudo ip address add dev wg0 192.168.7.2/24
#umask 077
#wg genkey | tee privatekey | wg pubkey > publickey
#cat publickey
sudo wg set wg0 private-key ./privatekey peer LYhSr2Q3mQcEVijh/nHZ44zk8SWQIqnYM7X8ikyw+D0= allowed-ips 192.168.7.0/24 endpoint 3.82.199.162:51820 persistent-keepalive 30
sudo wg showconf wg0
sudo ip link set up dev wg0
#wg show
