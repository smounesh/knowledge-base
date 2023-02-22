
# Openvpn-server and PKI setup:

1. Openvpn is used to create  secure ssl/tls vpn tunnels which provies secure access to remote servers and other programs running on those servers
2. It largely reduces ssh brute broke attacks by droping he connection request at initial phase 
3. Create own certificate authority using easy-rsa for issuing certs to clients and servers
4. This note is for setting up Openvpn on bastion and access other servers via bastion. Tested on OCI vm's
5. As a best practice the CA is created and Openvpn server is created on different servers but for simplicity ank lack of resource availability we create both on the same server

## Openvpn tunnel formation
```ASCII
+------------+---------------+                              +-----------------+-----------+
|            |public ip      |                              |  Public ip of   |           |
|            |of ovpn server |                              |  openvpn client |           |
|            +---------------+------------------------------+-----------------+           |
|                |                                                        |               |
|    Openvpn     |+------------------------------------------------------+|   Openvpn     |
|    server      |                  Openvpn private tunnel                |   client      |
|                |+------------------------------------------------------+|               |
|                |                                                        |               |
|                +----------+------------------------------------+-------+|               |
|                |          |           Public vpn tunnel        |        |               |
|                |          +------------------------------------+        |               |
|                |                                                        |               |
+----------------+                                                        +---------------+	
``` 

## Openvpn server and pki setup

1. This setup is tested on ubuntu 22.04, 20.04 and ubuntu based lxc containers

2. Install Openvpn
```sh
apt install openvpn -y
```
3. Download  `easy-rsa` from [github](https://github.com/OpenVPN/easy-rsa/releases) and place it in `/etc/openvpn/` folder 
```sh
wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.1.0/EasyRSA-3.1.0.tgz -O /tmp/easy-rsa; tar xzvf /tmp/easy-rsa; mv /tmp/EasyRSA-3.1.0 /etc/openvpn/easy-rsa
```
4. Create pki using easy-rsa 
```bash
./easyrsa init-pki
```
5. Add the following to the pki/vars file. 
```sh
set_var EASYRSA_REQ_COUNTRY    "INDIA"
set_var EASYRSA_REQ_PROVINCE   "Tamilnadu"
set_var EASYRSA_REQ_CITY       "Tirupur"
set_var EASYRSA_REQ_ORG        "Smounesh"
set_var EASYRSA_REQ_EMAIL      "shankar@smounesh.in"
set_var EASYRSA_REQ_OU         "smounesh Infra Team"
```
6. Build CA without paspharse
```sh
./easyrsa build-ca nopass
```
```sh
./easy-rsa gen-dh
```
7. Req  key for a Openvpn server without passphrase
```sh
./easyrsa gen-req ovpn-server nopass
```
8. Sign the key of Openvon server as a server with CA
```sh
./easyrsa sign-req server ovpn-server
```
9. Request a key for a client without passphrase
```sh
./easyrsa gen-req mounesh-laptop nopass
```
10. SIgn the key of clients as client with CA.
```sh
./easyrsa sign-req client mounesh-laptop
```
11. Run the safe ssl cmd
```sh
./easyrsa make-safe-ssl
```
12. Generate the ta.key file using the cmd which is essential to block brute-force attacks 
```sh
openvpn --genkey secret ta.key
```
13. Copy the ca.crt, dh.pem ovpn-server.crt and ovpn-server.key from /etc/openvpn-easy-rsa/pki to the /etc/openvpn folder. ovpn-server.key and ovpn-server.crt  can be found inside the pki/issued and pki/private folder 

14. Stop the openvpn and create config file for Openvpn server
```sh
systemctl stop openvpn
```
```sh
# OpenVPN config
# Filename: /etc/openvpn/server.conf

# Authentication is using client certs, user/pass and 2FA.
# Further, certificate common_name and username are pinned, to prevent
# users from sharing certificates.

# Ubuntu systemd runs openvpn with restricted capabilities and cannot run
# client connect/disconnect scripts to send email notifications. To fix:
# in /lib/systemd/system/openvpn@.service uncomment the LimitNPROC line
# and increase the default value from 10 to 100 processes
# Run systemctl daemon-reload and service openvpn restart


port 1194
proto udp
dev tun


dh dh.pem
ca ca.crt
tls-crypt ta.key
cert ovpn-server.crt
key  ovpn-server.key

server 10.8.0.0 255.255.255.0
topology subnet

ifconfig-pool-persist ipp.txt

# Route OCI-SF site-to-site via the tunnel
route 10.10.0.0 255.255.0.0
# Route OCI-HYD site-to-site via the tunnel
route 10.20.0.0 255.255.0.0

# Set this to internal network
#push "route 172.30.0.0 255.255.0.0"
#push "route 10.20.0.0  255.255.0.0"

# Set this if internal DNS is to be used
#push "dhcp-option DNS 10.10.4.51"
#push "dhcp-option DNS 10.10.4.52"

# Permit mutiple clients with same client cert
# duplicate-cn

keepalive 10 120

# Notify the client when the server restarts so it can auto reconnect
explicit-exit-notify 1

cipher AES-256-CBC
data-ciphers 'AES-256-CBC'
data-ciphers-fallback 'AES-256-CBC'

auth SHA256

max-clients 100
user nobody
group nogroup

persist-key
persist-tun

status openvpn-status.log
verb 3

client-config-dir ccd

# Authentiate users with both password and google-authenticator 2FA
# Clients have to enter password and 2fa token in the password prompt
#plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn

# Email root on user connct/disconnect
#script-security 2
#client-connect    /etc/openvpn/client-connect.sh
#client-disconnect /etc/openvpn/client-disconnect.sh

# Ensure username and certificate name are same, users cannot share certs
#auth-user-pass-verify /etc/openvpn/auth-cn-user.sh via-env
```
14. Allow ipv4 traffic forwarding 
```sh
# Filename: /etc/sysctl.d/99-enable-ipv4.conf
# Purpose: enable ipv4 forwarding to access other servers routed via bastion
net.ipv4.ip_forward=1
```
15. `Note:` If you provisioned vm's on OCI disable src/des check on bastion vnic and add a route to the subnet that you want to access via bastion using bastion vnic ip

16. Make a ipp.txt file which assigns static ip to the openvpn clients and servers
```sh
mounesh-laptop,10.8.0.2,
pve,10.8.0.3,
```
17. Create a ccd directory where we put the routes for other Openvpn servers
```sh
# Oracle Cloud Hyderabad VCN subnet
iroute 10.20.0.0 255.255.0.0
```
18. Sample client.conf.template file. Place this file in /etc/openvpn with corresponding values so can be used later when issuing certs to the clients. Windows clients uses a file extension client.ovpn 
```sh
; Smounesh OpenVPN client config

client

dev tun
proto udp

remote customs.smounesh.in

tls-crypt [inline]

resolv-retry infinite
nobind

persist-key
persist-tun

mute-replay-warnings

; auth-user-pass
auth-nocache
reneg-sec 0

remote-cert-tls server
cipher AES-256-CBC
data-ciphers 'AES-256-CBC'
data-ciphers-fallback 'AES-256-CBC'

auth SHA256

verb 3

<ca>
-----BEGIN CERTIFICATE-----
MIIDaTCCAlGgAwIBAgIUNyqFBam2F0ezEAZv0ZrlZfHAyfAwDQYJKoZIhvcNAQEL
BQAwIDEeMBwGA1UEAwwVY3VzdG9tcy5kZXYub21ueWsuY29tMB4XDTIyMDgyMzA2
MDQ1MloXDTMyMDgyMDA2MDQ1MlowIDEeMBwGA1UEAwwVY3VzdG9tcy5kZXYub21u
eWsuY29tMIIsajksBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqIhab14wPiDo
y/H6eFE174HCzxTsDYO9r/grCvt1MqtNszwimmAwM0V55ir3zMpokdIZLhGuMS9m
oVwDPkVxAAIa/Z0nMuzdt8oRw5GdkI+W+ajsaxqjleNkOr2W0q0H96kIvUXVN5bd
jdncJTLsV3wfgQUL2+g0o0a9UscO/SxJs/NIVcz2lFrqPvKyy0d+I9IQ0gBV7UUA
YxZMie7BMd8wKY6j7A2QGelI9+ZBVghosjrmxd47IclNMQUB9qjiYsRnoo0mpogi
CVnf2SdsSaVeknAUx5jidPYUjCjPS9aKs5txtuX7pbZ2ObFLRKU17HBLjhzPVD76
Z5bxqcSfgwIDAQABo4GaMIGXMAwGA1UdEwQFMAMBAf8wHQYDVR0OBBYEFDHHWJSn
rpfH69u85y9ppwcWHpyQMFsGA1UdIwRUMFKAFDHHWJSnrpfH69u85y9ppwcWHpyQ
oSSkIjAgMR4wHAYDVQQDDBVjdXN0b21zLmRldi5vbW55ay5jb22CFDcqhQWpthdH
sxAGb9Ga5WXxwMnwMAsGA1UdDwQEAwIBBjANBgkqhkiG9w0BAQsFAAOCAQEAJLA9
aT0j6Cn3sDi9xywsntIzzoCuSn5B6V08TTPgjoKfFzUEQA4R2qUF6z9lzBFis5qj
gxFKK98/3q4bykX/X6QKhLe+rnH43GZIArahArXjiYCaQqj9v8hZ9k88F4o5G2sI
kKscnGS3uhqacueVznkhnnJ941aCuKiaboqPB37ybuRs5PIq22QDm2duz7Q16UOi
TgJUIKvVpI7nODZebFo+HSvfeHufcOaatcR+Qlxe8Z9gslFnk1PGblZn9dT41xZB
0fgdrU2It4vKp5eRP/oCSkoXHY3fdqycGxm3YpEzbyUDfhx9W9tG3lfDGl9asTnm
VvTMdCNMz5cpi4Z0AA==
-----END CERTIFICATE-----
</ca>

<cert>
TODO: client certificate
</cert>

<key>
TODO: client key
</key>

<tls-crypt>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
e153710603092f0c452f2684bb3cc255
8121efc81479148d0f0f90f65fa5bce7
47b09f5a9beba815c922a6ba44ec03e3
1ce366bb2e8cf3d5b2ad8929120cac63
29599afd209304bd2d716239501dfbac
cfc529c6750518ca79552ddfbaf82cb0
bc3611c80cd96beca7f8ac25417b59b7
504524985281dskc2aa454847887ead2a94
79b3581744eb57e7c952d213b8d9ce33
d0707d92ad6458812709aa445d66c17c
455f33378f8f6e89952cea368d1fe068
39083b167320974152c81cf03af5924b
712e35861ade9c29c8e83c45b943fd63
fa57f097d251ff9cbd7ae2421b00e2ee
218fc695bb6ebe4c853824e2b410f798
60a413d0b9c687a5265f23f95b400d46
-----END OpenVPN Static key V1-----
</tls-crypt>
```
19. Create a dns record for the public ip of your bastion for the value you put in the remote session of the client.ovpn or client.conf file

20. Start the openvpn and set to start on boot
```sh
systemctl enable --now openvpn
```
21. If everything works well you will see a new bridge adapter in 10.8.0.1/24 segment
```sh
ip a
```
## References:
* [https://community.openvpn.net/openvpn/wiki/HOWTO#Linux](https://community.openvpn.net/openvpn/wiki/HOWTO#Linux)
* [https://github.com/OpenVPN/easy-rsa/blob/master/README.quickstart.md](https://github.com/OpenVPN/easy-rsa/blob/master/README.quickstart.md)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-22-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-22-04)

## TODO:
* Check easy-rsa make-safe-ssl command 