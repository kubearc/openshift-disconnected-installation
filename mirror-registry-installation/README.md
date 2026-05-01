# Registry Node Configuration

> The registry node requires **two active network interfaces**:
> - `192.168.122.20` — connected to the internet (for mirroring images from Red Hat)
> - `192.168.200.20` — connected to the isolated cluster network (for serving images to cluster nodes)

### Prerequisites

| Resource | Minimum | Recommended |
|:---------|:--------|:------------|
| RAM | 8 GB | 16 GB |
| vCPU | 4 | 4 |
| Storage | 100 GB | 200 GB |

> **Note on user permissions:**
> - If you are logged in **directly as root** (locally or via SSH), you will not encounter any environment-related issues and can proceed normally.
> - If you are logged in as a **standard user**, switch to root using `sudo su` — **not** `su` alone. Using plain `su` drops you into a root shell but does not carry over the environment variables that the mirror-registry installer depends on, which will cause the installation to fail silently.

---

### 2.1 Basic Network Configuration

Add the registry hostname to `/etc/hosts`:

```bash
echo "192.168.200.20  registry.example.com registry" >> /etc/hosts
```

Verify connectivity:

```bash
ping -c 2 registry.example.com
```

---

### 2.2 Download the Mirror Registry

Navigate to [console.redhat.com](https://console.redhat.com), log in with your Red Hat account, and go to:

**OpenShift → Downloads → OpenShift Disconnected → mirror-registry**

Download the `mirror-registry-amd64.tar.gz` tarball.

---

### 2.3 Extract and Install

If the tarball was downloaded as a standard user, copy it to the root working directory first:

```bash
cp /home/<your-username>/mirror-registry-amd64.tar.gz /root/
cd /root
tar -xvf mirror-registry-amd64.tar.gz
```

Install the mirror registry:

```bash
./mirror-registry install \
  --quayHostname registry.example.com \
  --quayRoot /mirror
```

> The installer will output the generated credentials (`init` username and a random password) at the end. **Save these immediately** — you will need them when building the combined pull secret for `install-config.yaml`.

---

### 2.4 Firewall and SELinux

```bash
# Open the registry port
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --reload

# Restore correct SELinux context on the registry storage directory
restorecon -RFv /mirror
```

---

### 2.5 Verify the Registry

**Via CLI — test podman login:**

```bash
podman login registry.example.com:8443 \
  --username init \
  --password <generated-password> \
  --tls-verify=false
```

**Via browser — open the Quay web UI:**

```
https://registry.example.com:8443
```

Log in with username `init` and the password from the installer output. You should see the Quay dashboard confirming the registry is running.

---

### 2.6 Trust the Registry Certificate on the Bastion Node

Run the following on the **bastion node** so that `oc` and `openshift-install` can communicate with the registry without TLS errors:

```bash
scp root@registry.example.com:/mirror/quay-rootCA/rootCA.pem \
  /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```

---

### 2.7 Mirror OCP Release Images

On the **registry node** (which has internet access), set the following environment variables:

```bash
export OCP_VERSION=4.18.9
export LOCAL_REGISTRY="registry.example.com:8443"
export LOCAL_REPOSITORY="ocp4/openshift4"
export PRODUCT_REPO="openshift-release-dev"
export RELEASE_NAME="ocp-release"
export ARCHITECTURE="x86_64"
```

Log in to both registries:

```bash
podman login registry.redhat.io
podman login ${LOCAL_REGISTRY}
```

Run the mirror command:

```bash
oc adm release mirror \
  -a pull-secret.txt \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_VERSION}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_VERSION}-${ARCHITECTURE}
```

> Save the `imageContentSources` block from the output — you will paste it into `install-config.yaml`.

---

## Phase 3 — Bastion: OCP Client Tools & Installation Config

### 3.1 Download and Install OCP Tooling

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.9/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.9/openshift-install-linux.tar.gz

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

---

### Skills Covered

`OpenShift` · `RHCOS` · `UPI` · `Disconnected / Air-gapped` · `HAProxy` · `BIND DNS` · `Mirror Registry` · `Linux`

---

*Kubearc Training | lab.example.com | For internal lab use only*
