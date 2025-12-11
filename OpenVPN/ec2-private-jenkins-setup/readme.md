# Private Jenkins Setup using OpenVPN Access Server (AWS)

This document describes the setup of a **private Jenkins server** placed inside a **private subnet**, accessed securely using an **OpenVPN Access Server** deployed in a public subnet.

The goal is to access Jenkins **only through VPN**, without exposing SSH or Jenkins UI to the internet.

---

## Network Topology Overview

Internet
   |
[Public Subnet]
   |-- OpenVPN Access Server (Public IP)
        - Ports: 22, 443, 943–945, 1194
        - VPN client network → access private subnet

[Private Subnet]
   |-- Jenkins Server (No Public IP)
        - Access via VPN only (SSH + Jenkins UI)

---

## Security Groups

### **OpenVPN Access Server SG**
| Port | Protocol | Source        | Purpose                |
|------|----------|----------------|-------------------------|
| 22   | TCP      | 0.0.0.0/0      | SSH                     |
| 443  | TCP      | 0.0.0.0/0      | Web Admin UI            |
| 943–945 | TCP  | 0.0.0.0/0       | OpenVPN UI/API          |
| 1194 | UDP      | 0.0.0.0/0      | VPN Tunnel              |

---

### **Jenkins Server SG**
> Only VPN clients should access Jenkins.

Assuming default OpenVPN client network:
172.27.224.0/20
| Port | Protocol | Source             | Purpose           |
|------|----------|---------------------|-------------------|
| 22   | TCP      | 172.27.224.0/20     | SSH via VPN       |
| 8080 | TCP      | 172.27.224.0/20     | Jenkins UI via VPN |


for this i used default thing here 
| Port | Protocol | Source             | Purpose           |
|------|----------|---------------------|-------------------|
| 22   | TCP      | 0.0.0.0/0     | SSH via VPN       |
| 8080 | TCP      | 0.0.0.0/0     | Jenkins UI via VPN |

---
## OpenVPN Routing Configuration

### **Add VPC CIDR to VPN routing**
## Step 5: Configure OpenVPN Routing
SSH into OpenVPN server and run:
```bash
# Route VPN clients to your VPC CIDR
sudo /usr/local/openvpn_as/scripts/sacli --key "vpn.server.routing.private_network.0" --value "<VPC_CIDR>" ConfigPut

# Route only VPC traffic through VPN
sudo /usr/local/openvpn_as/scripts/sacli --key "vpn.client.routing.reroute_gw" --value "false" ConfigPut

# Apply changes
sudo /usr/local/openvpn_as/scripts/sacli start

# Verify settings
sudo /usr/local/openvpn_as/scripts/sacli ConfigQuery | grep "vpn.server.routing"
```

for example 
```
# Add your VPC CIDR (10.0.0.0/16) to routing
sudo /usr/local/openvpn_as/scripts/sacli --key "vpn.server.routing.private_network.0" --value "10.0.0.0/16" ConfigPut

# Don't route all internet traffic through VPN (only route VPC traffic)
sudo /usr/local/openvpn_as/scripts/sacli --key "vpn.client.routing.reroute_gw" --value "false" ConfigPut

# Apply changes and restart OpenVPN
sudo /usr/local/openvpn_as/scripts/sacli start

# Verify the settings were applied
sudo /usr/local/openvpn_as/scripts/sacli ConfigQuery | grep "vpn.server.routing"
```
Expected output: ```"vpn.server.routing.private_network.0": "10.0.0.0/16" ```

---
## Accessing Private Jenkins

- Connect to VPN
- Accessing Jenkins (Private Subnet) ```ssh -i key.pem ubuntu@<private-ip>```
- Jenkins Web UI ```http://<private-ip>:8080```
