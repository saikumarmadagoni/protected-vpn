# OpenVPN with OAuth2 (Okta) Integration

This project demonstrates how to configure an OpenVPN server integrated with OAuth2 authentication using Okta. It enables secure browser-based authentication instead of traditional username/password login.

---

## Overview

This setup includes:

- OpenVPN server running on UDP port 1194
- OAuth2 authentication using Okta
- openvpn-auth-oauth2 for external authentication
- Full tunnel internet access through VPN
- Tunnelblick client configuration

---

## Architecture

Client (Tunnelblick) -> OpenVPN Server -> Management Interface -> openvpn-auth-oauth2 -> Okta

---

## Prerequisites

- Linux server (Ubuntu or Debian)
- OpenVPN 2.6 or later
- Public IP address
- Domain name for OAuth callback
- Okta tenant with OAuth2 application configured

---

## OpenVPN Server Configuration

File: /etc/openvpn/server/server.conf

port 1194
proto udp
dev tun

topology subnet
server 10.8.0.0 255.255.255.0

ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/server.crt
key /etc/openvpn/pki/server.key
crl-verify /etc/openvpn/pki/crl.pem

tls-auth /etc/openvpn/pki/ta.key 0

persist-key
persist-tun

keepalive 10 120
cipher AES-256-GCM
verb 3
dh none

management /run/openvpn/server.sock unix /etc/openvpn/password.txt
management-client-auth
auth-user-pass-verify /bin/true via-env
username-as-common-name

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

---

## Management Password

Create file: /etc/openvpn/password.txt

mysecretpassword

chmod 600 /etc/openvpn/password.txt

---

## OAuth2 Configuration

File: /etc/openvpn-auth-oauth2/config.yaml

http:
  baseurl: "https://your-domain"
  listen: ":9000"

oauth2:
  provider: "generic"
  client:
    id: "<client-id>"
    secret: "<client-secret>"
  endpoint:
    auth: "https://your-okta-domain/oauth2/v1/authorize"
    token: "https://your-okta-domain/oauth2/v1/token"
  issuer: "https://your-okta-domain/oauth2/default"
  scopes:
    - openid
    - profile
    - email

validate:
  groups: []

openvpn:
  addr: "unix:///run/openvpn/server.sock"
  password: "file:///etc/openvpn/password.txt"

---

## Start Services

systemctl enable openvpn-server@server
systemctl start openvpn-server@server

systemctl enable openvpn-auth-oauth2
systemctl start openvpn-auth-oauth2

---

## Enable Internet Access

sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens4 -j MASQUERADE

apt install iptables-persistent -y
netfilter-persistent save

---

## Client Configuration

client
dev tun
proto udp
remote <SERVER_IP> 1194

resolv-retry infinite
nobind
persist-key
persist-tun

remote-cert-tls server
cipher AES-256-GCM
verb 3

key-direction 1

Include certificates in blocks (<ca>, <cert>, <key>, <tls-auth>)

Remove auth-user-pass

---

## Authentication Flow

1. Client connects to OpenVPN
2. OAuth service handles authentication
3. Browser redirects to Okta
4. User logs in
5. Authentication is validated
6. VPN connects successfully

---

## Troubleshooting

Check OpenVPN:

systemctl status openvpn-server@server

Check OAuth:

systemctl status openvpn-auth-oauth2
journalctl -u openvpn-auth-oauth2 -f

---

References : 

<img width="2972" height="1502" alt="image" src="https://github.com/user-attachments/assets/21ea6265-a9fc-4d77-9a0e-722b3dd99867" />




## Author

Sai Goud
Cloud Engineer
