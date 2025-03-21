## OpenVPN Installation
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y

make-cadir ~/openvpn-ca
cd ~/openvpn-ca

nano vars
```sh
set_var EASYRSA_REQ_COUNTRY    "US"
set_var EASYRSA_REQ_PROVINCE   "NY"
set_var EASYRSA_REQ_CITY       "New York"
set_var EASYRSA_REQ_ORG        "MyOrg"
set_var EASYRSA_REQ_EMAIL      "admin@example.com"
set_var EASYRSA_REQ_OU         "Community"
```

## Generate Server Config

./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key

sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem ta.key /etc/openvpn/server/


sudo vi /etc/openvpn/server.conf
```sh
port 1194
proto udp
dev tun

ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem
tls-auth /etc/openvpn/server/ta.key 0

server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

key-direction 0
cipher AES-256-CBC
auth SHA256

keepalive 10 120
persist-key
persist-tun

user nobody
group nogroup

status openvpn-status.log
verb 3
```

### Allow IP forwarding

sudo vi /etc/sysctl.conf
```sh
uncomment
net.ipv4.ip_forward=1
```
sudo sysctl -p



```
sudo chown root:root /etc/openvpn/server/*
sudo chmod 600 /etc/openvpn/server/*.key
```


sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
sudo iptables-save | sudo tee /etc/iptables.rules
sudo apt install iptables-persistent -y
sudo netfilter-persistent save

sudo systemctl restart openvpn@server
sudo systemctl enable openvpn@server


## Client generation
cd ~/openvpn-ca
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1


sudo vi ~/openvpn-ca/make_client_config.sh

```sh
#!/bin/bash

KEY_DIR=~/openvpn-ca/pki
OUTPUT_DIR=~/client-configs
BASE_CONFIG=~/openvpn-ca/base.conf

mkdir -p ${OUTPUT_DIR}

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/issued/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/private/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ~/openvpn-ca/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn

echo "Generated: ${OUTPUT_DIR}/${1}.ovpn"
```
sudo chmod +x ~/openvpn-ca/make_client_config.sh

sudo vi ~/openvpn-ca/base.conf
```sh
client
dev tun
proto udp
remote YOUR_SERVER_IP 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
verb 3
key-direction 1
```
cd ~/openvpn-ca
./make_client_config.sh client1


## Transfer file

scp remoteServer:~/client-configs/client1.ovpn ./client1.ovpn