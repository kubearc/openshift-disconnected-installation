# OpenShift Container Platform — Disconnected UPI Installation
### Kubearc Training Series | Lab Guide v1.0

---

> **About This Guide**
> This lab guide walks you through a User Provisioned Infrastructure (UPI) installation of OpenShift Container Platform in a **fully disconnected (air-gapped) environment**. Internet access is restricted to a dedicated registry node only. All other cluster nodes operate on an isolated network with no direct internet connectivity.

---

## Lab Architecture

### Network Design

| Network | Subnet | Purpose |
|:--------|:-------|:--------|
| Isolated (Cluster) Network | `192.168.200.0/24` | Bastion, Bootstrap, Masters |
| Internet-Connected Network | `192.168.122.0/24` | Registry node only |

> The registry node sits on **both** networks — it pulls images from the internet via `192.168.122.0/24` and serves them to the cluster nodes via `192.168.200.0/24`.

### Node Inventory

| Hostname | IP Address | Role | Minimum Hardware |
|:---------|:-----------|:-----|:-----------------|
| `service.example.com` | `192.168.200.10` | Bastion / Helper Node | 4 vCPU · 16 GB RAM · 60 GB HDD |
| `registry.example.com` | `192.168.200.119` (isolated) · `192.168.122.119` (internet) | Mirror Registry | 4 vCPU · 16 GB RAM · 200 GB HDD |
| `bootstrap.example.com` | `192.168.200.30` | Bootstrap Node | 4 vCPU · 16 GB RAM · 60 GB HDD |
| `master1.example.com` | `192.168.200.41` | Control Plane Node | 4 vCPU · 16 GB RAM · 60 GB HDD |
| `master2.example.com` | `192.168.200.42` | Control Plane Node | 4 vCPU · 16 GB RAM · 60 GB HDD |
| `master3.example.com` | `192.168.200.43` | Control Plane Node | 4 vCPU · 16 GB RAM · 60 GB HDD |

---

## Phase 1 — Bastion Node Configuration

The bastion node (`service.example.com`) acts as the central helper for the cluster. It hosts DNS, load balancing, and the ignition file HTTP server.

### 1.1 Install Required Services

```bash
yum install -y httpd bind bind-utils haproxy
```

---

### 1.2 Configure DNS (BIND / Named)

**Adjust the named listener and allow-query settings:**

```bash
ip=$(ip addr show | grep -oE 'inet ([0-9]*\.){3}[0-9]*' | awk '$2 !~ /^127\./ {print $2}')
sed -i "s/listen-on port 53 { 127.0.0.1; };/listen-on port 53 { $ip; };/g" /etc/named.conf

ip_with_suffix=$(ip addr show | grep -v '127.0.0.1' | awk '/inet / {split($2, a, "/"); split(a[1], b, "."); print b[1]"."b[2]"."b[3]".0/"a[2]}')
sed -i "s/allow-query     { localhost; };/allow-query     { ${ip_with_suffix//\//\\/}; };/g" /etc/named.conf
```

**Add zone declarations to `/etc/named.conf`:**

```bash
vim /etc/named.conf
```

```
zone "example.com" IN {
    type master;
    file "forward";
};

zone "200.168.192.in-addr.arpa" IN {
    type master;
    file "reverse";
};
```

---

**Create the forward zone file:**

```bash
cp /var/named/named.localhost /var/named/forward
vim /var/named/forward
```

```
$TTL 1D
@   IN SOA  service.example.com. root.example.com. (
                        2024010101  ; serial
                        1D          ; refresh
                        1H          ; retry
                        1W          ; expire
                        3H )        ; minimum

@               IN  NS      service.example.com.
@               IN  A       192.168.200.10

; Bastion / Helper
service.example.com.            IN  A   192.168.200.10
service.lab.example.com.        IN  A   192.168.200.10

; Registry (isolated network interface)
registry.example.com.           IN  A   192.168.200.119
registry.lab.example.com.       IN  A   192.168.200.119

; OCP API endpoints (load-balanced via Bastion)
api.lab.example.com.            IN  A   192.168.200.10
api-int.lab.example.com.        IN  A   192.168.200.10
*.apps.lab.example.com.         IN  A   192.168.200.10

; Cluster nodes
bootstrap.lab.example.com.      IN  A   192.168.200.30
master1.lab.example.com.        IN  A   192.168.200.41
master2.lab.example.com.        IN  A   192.168.200.42
master3.lab.example.com.        IN  A   192.168.200.43
```

---

**Create the reverse zone file:**

```bash
cp /var/named/forward /var/named/reverse
vim /var/named/reverse
```

```
$TTL 1D
@   IN SOA  service.example.com. root.example.com. (
                        2024010101  ; serial
                        1D          ; refresh
                        1H          ; retry
                        1W          ; expire
                        3H )        ; minimum

@       IN  NS      service.example.com.
@       IN  PTR     example.com.

; Bastion
10      IN  PTR     service.example.com.
10      IN  PTR     service.lab.example.com.
10      IN  PTR     api.lab.example.com.
10      IN  PTR     api-int.lab.example.com.

; Registry (isolated interface)
119      IN  PTR     registry.example.com.
119      IN  PTR     registry.lab.example.com.

; Cluster nodes
30      IN  PTR     bootstrap.lab.example.com.
41      IN  PTR     master1.lab.example.com.
42      IN  PTR     master2.lab.example.com.
43      IN  PTR     master3.lab.example.com.
```

**Set correct group ownership and start the service:**

```bash
chgrp named /var/named/forward
chgrp named /var/named/reverse
systemctl enable named --now
```

**Verify DNS resolution:**

```bash
dig api.lab.example.com
dig -x 192.168.200.10
```

---

### 1.3 Configure HAProxy (Load Balancer)

HAProxy handles traffic distribution across the control plane nodes. Replace `/etc/haproxy/haproxy.cfg` with the following:

```bash
vim /etc/haproxy/haproxy.cfg
```

```
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    option                  forwardfor except 127.0.0.0/8
    retries                 3
    maxconn                 20000
    timeout http-request    10000ms
    timeout http-keep-alive 10000ms
    timeout check           10000ms
    timeout connect         40000ms
    timeout client          300000ms
    timeout server          300000ms
    timeout queue           50000ms

#---------------------------------------------------------------------
# HAProxy Stats Dashboard
#---------------------------------------------------------------------
listen stats
    bind :9000
    stats uri /stats
    stats refresh 10000ms

#---------------------------------------------------------------------
# Kube API Server — port 6443
#---------------------------------------------------------------------
frontend k8s_api_frontend
    bind :6443
    mode tcp
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    balance source
    server  bootstrap   192.168.200.30:6443 check
    server  master1     192.168.200.41:6443 check
    server  master2     192.168.200.42:6443 check
    server  master3     192.168.200.43:6443 check

#---------------------------------------------------------------------
# Machine Config Server — port 22623
#---------------------------------------------------------------------
frontend ocp_machine_config_server_frontend
    bind :22623
    mode tcp
    default_backend ocp_machine_config_server_backend

backend ocp_machine_config_server_backend
    mode tcp
    balance source
    server  bootstrap   192.168.200.30:22623 check
    server  master1     192.168.200.41:22623 check
    server  master2     192.168.200.42:22623 check
    server  master3     192.168.200.43:22623 check

#---------------------------------------------------------------------
# OCP Ingress HTTP — port 80
# (Layer 4 TCP mode; Ingress Controller handles Layer 7)
#---------------------------------------------------------------------
frontend ocp_http_ingress_frontend
    bind :80
    mode tcp
    default_backend ocp_http_ingress_backend

backend ocp_http_ingress_backend
    mode tcp
    balance source
    server  master1     192.168.200.41:80 check
    server  master2     192.168.200.42:80 check
    server  master3     192.168.200.43:80 check

#---------------------------------------------------------------------
# OCP Ingress HTTPS — port 443
#---------------------------------------------------------------------
frontend ocp_https_ingress_frontend
    bind *:443
    mode tcp
    default_backend ocp_https_ingress_backend

backend ocp_https_ingress_backend
    mode tcp
    balance source
    server  master1     192.168.200.41:443 check
    server  master2     192.168.200.42:443 check
    server  master3     192.168.200.43:443 check
```

**Enable SELinux boolean and start HAProxy:**

```bash
setsebool -P haproxy_connect_any on
systemctl enable haproxy --now
```

**Disable and stop firewalld:**

```bash
systemctl disable firewalld --now
```

---


## Phase 2 — Bastion: OCP Client Tools & Installation Config

### 3.1 Download and Install OCP Tooling

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.21/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.21/openshift-install-linux.tar.gz

tar -xvf openshift-client-linux.tar.gz -C /usr/bin
tar -xvf openshift-install-linux.tar.gz -C /usr/bin
```

Verify the installed versions:

```bash
oc version
kubectl version
openshift-install version
```

---

### 3.2 Generate SSH Key Pair

This key will be embedded in the ignition configs to allow SSH access to cluster nodes:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ocp_key -N ""
```

---

### 3.3 Create `install-config.yaml`

```bash
mkdir ~/ocp4 && cd ~/ocp4
vim install-config.yaml
```

```yaml
apiVersion: v1
baseDomain: example.com
compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0                   # No workers in a compact/masters-only cluster
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: lab                       # This becomes the cluster name: lab.example.com
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths": ...}'      # Replace with your combined pull secret
sshKey: "ssh-ed25519 AAAA..."     # Replace with contents of ~/.ssh/ocp_key.pub
additionalTrustBundle: |          # Add the registry CA cert here
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
imageContentSources:              # Paste the block output by the mirror command
  - mirrors:
      - registry.example.com:8443/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
      - registry.example.com:8443/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

---

### 3.4 Generate Manifests and Ignition Configs

> **Important:** Back up `install-config.yaml` before running these commands — the file is consumed and deleted during manifest generation.

```bash
cp ~/ocp4/install-config.yaml ~/install-config.yaml.bak

openshift-install create manifests --dir=/root/ocp4
openshift-install create ignition-configs --dir=/root/ocp4
```

---

### 3.5 Configure Apache HTTP Server (Ignition File Server)

```bash
mkdir /var/www/html/ocp4
cp /root/ocp4/*.ign /var/www/html/ocp4/
restorecon -RFv /var/www/html/
chgrp -R apache /var/www/html/ocp4

# Change the default listen port to 8080 (avoid conflict with HAProxy port 80)
sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf
firewall-cmd --permanent --add-port=8080/tcp
# Allow httpd to listen on 8080 (not in default allowed list)
semanage port -a -t http_port_t -p tcp 8080
systemctl enable httpd --now
```

Verify the ignition files are served:

```bash
curl http://192.168.200.10:8080/ocp4/bootstrap.ign | python3 -m json.tool | head
```

---

## Phase 4 — Cluster Node Installation

### 4.1 Bootstrap Node

Power on the bootstrap VM using the **RHCOS (CoreOS) ISO**. When the live environment loads:

1. Configure the network interface with the correct static IP (`192.168.200.30`), gateway, and DNS (`192.168.200.10`).
2. Activate the connection and verify DNS resolution:

```bash
ping service.example.com
```

3. Apply the ignition configuration and install to disk:

```bash
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.200.10:8080/ocp4/bootstrap.ign \
  --insecure-ignition \
  --copy-network
```

4. Reboot the node:

```bash
sudo reboot
```

---

### 4.2 Control Plane Nodes (master1, master2, master3)

Repeat the same procedure for each master node. The steps are identical except:

- Assign each node its respective static IP (`192.168.200.41`, `.42`, `.43`)
- Use the **master ignition URL** instead:

```bash
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.200.10:8080/ocp4/master.ign \
  --insecure-ignition \
  --copy-network
```

Reboot each node after applying ignition:

```bash
sudo reboot
```

> Power on and complete nodes **one at a time** before moving on to the next.

---

## Phase 5 — Monitor Installation Progress

Once all nodes have been rebooted, the installation begins automatically. Monitor progress from the bastion node.

**Watch bootstrap progress:**

```bash
ssh -i ~/.ssh/ocp_key core@192.168.200.30
journalctl -b -f -u bootkube.service
```

**Monitor via HAProxy stats (browser):**

```
http://192.168.200.10:9000/stats
```

**Use the openshift-install command for progress tracking:**

```bash
openshift-install --dir=/root/ocp4 wait-for bootstrap-complete --log-level=info
```

Once bootstrap is complete, shut down the bootstrap VM and remove it from the HAProxy backend pools. Then wait for the installation to fully complete:

```bash
openshift-install --dir=/root/ocp4 wait-for install-complete --log-level=info
```

---

## Reference

### Official Documentation

- [OCP UPI Bare Metal Installation](https://docs.openshift.com/container-platform/4.18/installing/installing_bare_metal/installing-bare-metal.html)
- [Mirroring Images for Disconnected Installation](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/installing-mirroring-installation-images.html)

### Port Reference

| Port | Protocol | Component |
|:-----|:---------|:----------|
| 6443 | TCP | Kubernetes API Server |
| 22623 | TCP | Machine Config Server |
| 80 | TCP | Ingress HTTP |
| 443 | TCP | Ingress HTTPS |
| 8080 | TCP | Bastion HTTP (ignition files) |
| 8443 | TCP | Mirror Registry (Quay) |
| 9000 | TCP | HAProxy Stats |

Configure Firewall
```bash
# DNS
firewall-cmd --permanent --add-service=dns
# HAProxy ports
firewall-cmd --permanent --add-port=6443/tcp    # Kube API
firewall-cmd --permanent --add-port=22623/tcp   # Machine Config Server
firewall-cmd --permanent --add-port=80/tcp      # Ingress HTTP
firewall-cmd --permanent --add-port=443/tcp     # Ingress HTTPS
firewall-cmd --permanent --add-port=9000/tcp    # HAProxy Stats

firewall-cmd --reload
```

---

### Skills Covered

`OpenShift` · `RHCOS` · `UPI` · `Disconnected / Air-gapped` · `HAProxy` · `BIND DNS` · `Mirror Registry` · `Linux`

---

*Kubearc Training | lab.example.com | For internal lab use only*
