Small K3s Cluster on Two Laptops Using OPNsense and VLAN 10

![Proxmox](https://img.shields.io/badge/Proxmox-ED1C24?style=for-the-badge&logo=proxmox&logoColor=white) 
![OPNsense](https://img.shields.io/badge/OPNsense-EF4F1A?style=for-the-badge&logo=opnsense&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![K3s](https://img.shields.io/badge/K3s-339933?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Containerd](https://img.shields.io/badge/Containerd-0A0A0A?style=for-the-badge&logo=containerd&logoColor=white)
![MetalLB](https://img.shields.io/badge/MetalLB-FFAA00?style=for-the-badge&logo=metallb&logoColor=white)
![Calico](https://img.shields.io/badge/Calico-253B80?style=for-the-badge&logo=calico&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white)


Hardware Overview
Node	Model	CPU	RAM	Disk
Node 1	Gateway NV55C	Intel i5-460M, 2C/4T	8GB DDR3	250GB SSD
Node 2	Toshiba S55T-B	Intel i7-4710Q, 4C/8T	16GB DDR3L	500GB SSD
Switch	Netgear GS108Ev4	8-port Managed	VLAN-capable	Note: limited CARP support

Consider a more robust switch if full HA or advanced VLAN experiments are needed.

Phase 1 – Prepare Each Laptop

Install Debian Server on both laptops.

Set static IPs for management VLAN (10.0.10.0/24).

Install monitoring exporters: Prometheus Exporter, smartctl Exporter (RAM, CPU, drive temps).

Install Docker & Docker Compose

sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
sudo docker run hello-world

sudo apt-get install docker-compose-plugin


Install K3s

sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s
sudo k3s kubectl get nodes


To join additional agent nodes:

curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -


Firewall: Disable ufw for simplicity

sudo ufw disable


Or allow required ports:

ufw allow 6443/tcp      # Kubernetes API server
ufw allow from 10.42.0.0/16 to any   # Pods
ufw allow from 10.43.0.0/16 to any   # Services


Install Tailscale VPN for remote access

curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale up --ssh

Phase 2 – OPNsense & VLAN 10 Setup
Network Diagram
             +---------------------+
             |      Internet       |
             |  WAN 192.168.0.x   |
             +----------+----------+
                        |
                 [Switch Ports 2-5]
                        |
               +-----------------------+
               |   Proxmox Host        |
               | vmbr0 -> WAN (enp3s0)|
               | 192.168.0.218/24      |
               | Gateway: 192.168.0.1  |
               |                       |
               | vmbr1 -> LAN/VLAN10   |
               | vlan-aware             |
               +----------+------------+
                          |
         Switch Ports 3, 6, & 7 untagged VLAN10 (PVID 10)
                          |
   +------------------+-----------+------------------+
   |                  |           |                  |
debBlue Laptop     debGold Laptop  OPNsense VM
10.0.10.70/24     10.0.50/24      10.0.10.60/24
GW: 10.0.10.1     GW: 10.0.10.1   GW: 10.0.10.1
DNS: 10.0.10.1    DNS: 10.0.10.1  DNS: 10.0.10.1

Switch VLAN Configuration
VLAN	Ports	Notes
1	1–5,8	Default, untagged
10	3,6,7	Management VLAN, untagged, PVID 10

Ports 6 & 7 excluded from VLAN 1 to avoid conflicts.

Proxmox Host Network Configuration (/etc/network/interfaces)
# WAN bridge
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.218/24
    gateway 192.168.0.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0

# VLAN-aware LAN bridge
auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp1s0f1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094


Do not set bridge-pvid in Proxmox; let the switch handle VLAN untagging/PVID.

OPNsense Configuration

LAN (vtnet1) = 10.0.10.1/24

WAN (vtnet0) = 192.168.0.186/24

Only VLAN 10 (Management) is required; VLAN 20 removed.

Laptop / VM IP Assignments
Device	IP	Gateway	DNS
debBlue	10.0.10.70	10.0.10.1	10.0.10.1, 1.1.1.1
debGold	10.0.10.50	10.0.10.1	10.0.10.1
Ubuntu VM	10.0.10.60	10.0.10.1	10.0.10.1
Lessons Learned / Gotchas

Proxmox resets bridge-pvid on restart; handle untagging at the switch instead.

Verify physical port VLAN membership and PVID before touching Proxmox bridge settings.

Exclude management VLAN ports from VLAN 1 to avoid conflicts.

Avoid mixing HA experiments with VLAN setup on underpowered switches; it causes most network headaches.

Always test connectivity (ping gateway + external IP) before proceeding to next steps.
