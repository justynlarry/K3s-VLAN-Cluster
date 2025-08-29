Small K3s Cluster on Two Laptops Using OPNsense and VLAN 10 -> Exposing Web-Page through a Cloudflare Tunnel

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


# Hardware Overview
# Node	  Model	            CPU	                    RAM	          Disk
Node 1	Gateway NV55C	    Intel i5-460M, 2C/4T	  8GB DDR3	    250GB SSD
Node 2	Toshiba S55T-B	  Intel i7-4710Q, 4C/8T	  16GB DDR3L	  500GB SSD
Switch	Netgear GS108Ev4	8-port Managed	        VLAN-capable	Note: limited CARP support

* Consider a more robust switch if full HA or advanced VLAN experiments are needed.

# Phase 1 – Prepare Each Laptop

1. Install Debian Server on both laptops.

2. Set static IPs for management VLAN (10.0.10.0/24).

3. Install monitoring exporters: Prometheus Exporter, smartctl Exporter (RAM, CPU, drive temps).

4. Install Docker & Docker Compose
```bash
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
```

5. Install K3s
```bash
sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

* To join additional agent nodes:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

6. Firewall: Disable ufw for simplicity
```bash
sudo ufw disable
```

* Or allow required ports:
```bash
ufw allow 6443/tcp      # Kubernetes API server
ufw allow from 10.42.0.0/16 to any   # Pods
ufw allow from 10.43.0.0/16 to any   # Services
```

7. Install Tailscale VPN for remote access
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale up --ssh
```

# Phase 2 – OPNsense & VLAN 10 Setup
Network Diagram
```
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
   |                             |                  |
debBlue Laptop             debGold Laptop        OPNsense VM
10.0.10.70/24              10.0.50/24            10.0.10.60/24
GW: 10.0.10.1              GW: 10.0.10.1         GW: 10.0.10.1
DNS: 10.0.10.1             DNS: 10.0.10.1        DNS: 10.0.10.1
```

# Switch VLAN Configuration
VLAN	Ports	  Notes
1	    1–5,8	  Default, untagged
10	  3,6,7	  Management VLAN, untagged, PVID 10

* Ports 6 & 7 excluded from VLAN 1 to avoid conflicts.

# Proxmox Host Network Configuration (/etc/network/interfaces)
```bash
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
```

* Do not set bridge-pvid in Proxmox; let the switch handle VLAN untagging/PVID.

# OPNsense Configuration

LAN (vtnet1) = ```10.0.10.1/24```
WAN (vtnet0) = ```192.168.0.186/24```
Only VLAN 10 (Management) is required; VLAN 20 removed.

# Laptop / VM IP Assignments
Device	    IP	          Gateway	DNS
debBlue	    10.0.10.70	   10.0.10.1	10.0.10.1, 1.1.1.1
debGold	    10.0.10.50	   10.0.10.1	10.0.10.1
Ubuntu VM	  10.0.10.60	   10.0.10.1	10.0.10.1




# Lessons Learned / Gotchas

- Proxmox resets bridge-pvid on restart; handle untagging at the switch instead.
- Verify physical port VLAN membership and PVID before touching Proxmox bridge settings.
- Exclude management VLAN ports from VLAN 1 to avoid conflicts.



# Setting Up the K3s Cluster On Two Computers:

On Primary (Where Master Node will be set up):

1. Set Environment Variables:

```Bash

export VIP_IP="10.0.10.90"
export NODE_IP="10.0.10.50"
export TOKEN="SuperSecretToken123"
export INTERFACE="enp8s0"
```

2. Install K3s Master:
- This command downloads and installs K3s with the custom flags required for a manual CNI (Calico) setup. It also initializes the cluster and sets the node-ip.

```Bash

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="\
  --cluster-init \
  --disable-network-policy \
  --flannel-backend=none \
  --disable servicelb \
  --disable traefik \
  --cluster-cidr=10.42.0.0/16 \
  --service-cidr=10.43.0.0/16 \
  --token $TOKEN \
  --tls-san $VIP_IP \
  --node-ip $NODE_IP \
  --node-external-ip $NODE_IP" \
  sh -s - server
```
3. Generate the kube-vip Manifest:
- This command uses the kube-vip Docker image to generate a YAML manifest file that we will manually apply.

```Bash

sudo docker run --network host --rm ghcr.io/kube-vip/kube-vip:v0.7.0 \
  manifest daemonset \
  --services \
  --controlplane \
  --arp \
  --interface $INTERFACE \
  --vip $VIP_IP \
  --taint \
  > /home/gold-user/kube-vip.yaml
```
4. Manually Apply the Manifest:
- This applies the kube-vip manifest directly to the running cluster, which ensures it is created. Wait 30 seconds for the pod to start.

```Bash

sudo kubectl apply -f /home/gold-user/kube-vip.yaml
```
5. Verify kube-vip is Running and the VIP is Active:

```Bash

# Check for the running pod (Status should be "Running")
sudo kubectl -n kube-system get pods -l app=kube-vip-cloud-provider

# Check that the VIP is listed on the interface
ip addr show $INTERFACE
```
* You should see inet 10.0.10.90 listed as an IP on the interface.

# Phase 2: CNI (Calico) Installation (debGold)
The cluster is now installed but is in a non-functional state, waiting for a network plugin. This is the expected and correct behavior. The Calico CNI will provide network connectivity for all pods.

On Primary:

1. Apply the Calico Manifest:

```Bash

sudo kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
2. Verify Master Node Status:
- Wait a minute or two for Calico to deploy its pods. The master node status should change to Ready.

```Bash

sudo kubectl get nodes
```

# Phase 3: Worker Node Installation (debBlue)
Now that the master node is fully configured and operational, you can connect the worker node. We will use the master's private host IP (10.0.10.50) instead of the VIP, which was a point of failure in our troubleshooting.

On Secondary (computer hosting worker node):

1. Set Environment Variables:

```Bash

export MASTER_IP="10.0.10.50"
export NODE_IP="10.0.10.70"
export TOKEN="SuperSecretToken123"
```

2. Install the K3s Agent:
This command joins the node to the cluster. Crucially, it includes the same ```--disable-network-policy``` and ```--flannel-backend=none``` flags as the master to avoid a critical configuration mismatch.

```Bash

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="\
  --server https://$MASTER_IP:6443 \
  --token $TOKEN \
  --node-ip $NODE_IP \
  --node-external-ip $NODE_IP \
  --disable-network-policy \
  --flannel-backend=none" \
  sh -s - agent
```

3. Verify Cluster Status:
Go back to the master node (Master) and check the status of both nodes.

```Bash

# On Master:
sudo kubectl get nodes
```

* Both debgold and debblue should now be in a Ready state.

# Phase 4: Ingress and Cloudflare Tunnel
This final phase sets up a secure and production-ready way to expose your applications without opening any ports on your firewall.

On debGold:

1. Install Nginx Ingress Controller:
This will handle all incoming HTTP/HTTPS traffic from the internet.

```Bash

sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm repo update
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=ClusterIP
```

2. Install and Configure the Cloudflare Tunnel:
This creates an encrypted tunnel from your master node to Cloudflare's network.

- Install cloudflared:

- Any Debian Based Distribution (other distributions can be found at: https://pkg.cloudflare.com/index.html )
```Bash
# Add cloudflare gpg key
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# Add this repo to your apt repositories
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# install cloudflared
sudo apt-get update && sudo apt-get install cloudflared
```
- Log in and create the tunnel (follow the on-screen instructions):

```Bash

cloudflared tunnel login
cloudflared tunnel create k3s-tunnel
```

Create the config file (```nano /etc/cloudflared/config.yml```):
(Replace ```<YOUR_TUNNEL_ID>``` with the ID from the previous command)


```YAML

tunnel: <YOUR_TUNNEL_ID>
credentials-file: /home/gold-user/.cloudflared/<YOUR_TUNNEL_ID>.json
ingress:
  - hostname: <CNAME>.<DOMAIN_NAME>
    service: http://10.0.10.50 # Route to the Nginx Ingress Controller on your master node
  - service: http_status:404
```

- Create DNS record and run as a service:

```Bash

cloudflared tunnel route dns k3s-tunnel <CNAME>.<DOMAIN_NAME>
sudo cloudflared --config /etc/cloudflared/config.yml service install
```

# Phase 5: Deploying Your PHP Web Page
This final step uses a single YAML file to deploy a simple PHP web page to your cluster. It uses a single container image that includes both Nginx and a PHP interpreter.

On Primary:

Create the YAML file (```nano php-web-simple.yaml```).
```YAML
---
# 1. The ConfigMap: This holds our simple PHP file.
apiVersion: v1
kind: ConfigMap
metadata:
  name: php-files-cm
data:
  index.php: |
    <?php
    echo "<h1>Hello from PHP!</h1>";
    echo "This page was served from the debGold and debBlue cluster.";
    echo "<br>";
    echo "The current time is " . date("Y-m-d H:i:s");
    ?>

---
# 2. The Deployment: Using a single image that combines Nginx and PHP.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: php-web
  template:
    metadata:
      labels:
        app: php-web
    spec:
      volumes:
        - name: php-code
          configMap:
            name: php-files-cm
      containers:
      - name: php-container
        image: webdevops/php-nginx:alpine-3.1
        ports:
          - containerPort: 80
        volumeMounts:
          - name: php-code
            mountPath: /app/index.php
            subPath: index.php
        env:
        - name: WEB_DOCUMENT_ROOT
          value: "/app"
        
---
# 3. The Service: Exposes our Deployment.
apiVersion: v1
kind: Service
metadata:
  name: php-web-service
spec:
  selector:
    app: php-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---
# 4. The Ingress: Exposes our Service to the world.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: php-web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: hello.sr-one.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: php-web-service
            port:
              number: 80
```

- Delete the old hello-web resources:

```Bash

sudo kubectl delete deployment hello-web-deployment
sudo kubectl delete service hello-web-service
sudo kubectl delete ingress hello-web-ingress
```

3. Apply the new manifest:

```Bash

sudo kubectl apply -f php-web-simple.yaml
```
- Your cluster will now pull the new image and deploy your PHP-powered website.

Adding a Second Website to the existing cluster

1. Create a new YAML file (e.g., blog-web.yaml) with a new set of resources for your second website. Use different names and a new hostname in the Ingress rule.

```YAML

---
# Deployment for your new blog
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog-web
  template:
    metadata:
      labels:
        app: blog-web
    spec:
      containers:
      - name: blog-container
        image: nginx:latest # We'll just use the default Nginx page again
        ports:
        - containerPort: 80
---
# Service for your new blog
apiVersion: v1
kind: Service
metadata:
  name: blog-service
spec:
  selector:
    app: blog-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
# Ingress to route traffic for blog.sr-one.net
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: <CNAME>.<DOMAIN_NAME>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```
2. Apply the new manifest, Don't delete anything; just add the new application.

```Bash

sudo kubectl apply -f blog-web.yaml
```
Add a new CNAME record to Cloudflare. Go into your Cloudflare DNS settings and add a new CNAME record for <CNAME>.<DOMAIN_NAME> that points to the same tunnel ID you created above.

* A Few Tips:
- Avoid mixing HA experiments with VLAN setup on underpowered switches; it causes most network headaches.
- Always test connectivity (ping gateway + external IP) before proceeding to next steps.
