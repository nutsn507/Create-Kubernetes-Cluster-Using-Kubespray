
# Kubernetes Cluster Implementation with Kubespray

This guide provides comprehensive instructions for deploying a production-ready, high-availability (HA) Kubernetes cluster using Kubespray. The setup involves configuring multiple nodes to serve as both master and worker nodes.

---

## Prerequisites

### Hardware Requirements

Ensure each node meets the following minimum requirements:

- **Control Plane Nodes**:
  - Memory: 2 GB or more
  - CPU: 2 cores or more

- **Worker Nodes**:
  - Memory: 1 GB or more
  - CPU: 2 cores or more

*Note*: Actual requirements may vary based on workload. For detailed sizing guidelines, refer to the official Kubernetes documentation.

### Software Requirements

- Operating System: Ubuntu 20.04 LTS or similar
- Control Machine:
  - Ansible v2.14+
  - Jinja 2.11+
  - Python-netaddr

*Note*: If running Kubespray from a non-root user account, ensure proper privilege escalation is configured on target servers.

### Network Requirements

- IPv4 Forwarding: Must be enabled on all target servers.
- Firewall: Firewalls are not managed by Kubespray. Disable or configure them appropriately to allow necessary traffic.
- Internet Access: Target servers require internet access to pull necessary Docker images. For offline environments, additional configuration is required.
- IP Addresses: Assign static IPs to all nodes.
  - Ansible Node: 10.255.0.50
  - knode01: 10.255.0.51
  - knode02: 10.255.0.52
  - knode03: 10.255.0.53

---

## Step 1: Prepare the Environment

1. **Disable Swap**:
   - Swap must be disabled because Kubernetes requires access to memory without potential interference from swap space. This ensures consistent performance and stability of the cluster.
   ```bash
   sudo swapoff -a
   ```

2. **Stop and Disable Firewall**:
   - **For POC (Proof of Concept) environments**:
     - The firewall can block essential communication between nodes. Disabling it during setup ensures there are no connectivity issues.
     ```bash
     sudo systemctl stop ufw
     sudo systemctl disable ufw
     ```
   - **For Production environments**:
     - It is recommended to configure the firewall to allow necessary traffic instead of disabling it. Below is a list of the required ports and the commands to allow them:
     ```bash
     # Allow Kubernetes API Server (port 6443)
     sudo ufw allow 6443/tcp

     # Allow etcd server client API (port 2379-2380)
     sudo ufw allow 2379:2380/tcp

     # Allow kubelet healthz (port 10250)
     sudo ufw allow 10250/tcp

     # Allow node communication ports (ports 30000-32767 for NodePort services)
     sudo ufw allow 30000:32767/tcp
     ```

     ### Control Plane Ports
     | Protocol | Direction | Port Range | Purpose                     | Used By              |
     |----------|-----------|------------|-----------------------------|----------------------|
     | TCP      | Inbound   | 6443       | Kubernetes API server       | All                  |
     | TCP      | Inbound   | 2379-2380  | etcd server client API      | kube-apiserver, etcd |
     | TCP      | Inbound   | 10250      | Kubelet API                 | Self, Control plane  |
     | TCP      | Inbound   | 10259      | kube-scheduler              | Self                 |
     | TCP      | Inbound   | 10257      | kube-controller-manager     | Self                 |

     Although etcd ports are included in the control plane section, you can also host your own etcd cluster externally or on custom ports.

     ### Worker Node Ports
     | Protocol | Direction | Port Range | Purpose             | Used By            |
     |----------|-----------|------------|---------------------|--------------------|
     | TCP      | Inbound   | 10250      | Kubelet API         | Self, Control plane|
     | TCP      | Inbound   | 10256      | kube-proxy          | Self, Load balancers|
     | TCP      | Inbound   | 30000-32767 | NodePort Services†  | All                |

     For a complete list of ports and their usage in Kubernetes, refer to the [Kubernetes documentation](https://kubernetes.io/docs/setup/best-practices/cluster-large/#check-required-ports).

3. **Enable IPv4 Forwarding**:
   - IPv4 forwarding must be enabled for networking between pods and services to function correctly in Kubernetes.
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

4. **Modify Hosts File**:
   - Update the `/etc/hosts` file on the Ansible Node to enable hostname resolution between the nodes. This ensures that nodes can communicate with each other using their hostnames instead of IP addresses.
   ```plaintext
   10.255.0.51  knode01
   10.255.0.52  knode02
   10.255.0.53  knode03
   ```

5. **Configure SSH Access**:
   - Set up SSH key-based authentication so the Ansible Node can connect to all other nodes without requiring a password. This is necessary for Ansible to manage the nodes during deployment.
   ```bash
   ssh-keygen -t rsa
   ssh-copy-id knode01
   ssh-copy-id knode02
   ssh-copy-id knode03
   ```

---

## Step 2: Install Ansible and Clone Kubespray

1. **Install Ansible and Dependencies**:
   - Ansible is required to run the Kubespray playbooks, which automate the deployment of the Kubernetes cluster. Installing the necessary dependencies ensures all required tools are available.
   ```bash
   sudo apt update
   sudo apt install python3-pip -y
   pip3 install --upgrade pip
   pip3 install ansible==2.14 jinja2==2.11 netaddr
   ```

2. **Clone the Kubespray Repository**:
   - The Kubespray repository contains the playbooks and scripts needed for deploying Kubernetes. Cloning the repository provides access to the latest version.
   ```bash
   git clone https://github.com/kubernetes-sigs/kubespray.git
   cd kubespray
   pip3 install -r requirements.txt --default-timeout=1000
   ```

---

## Step 3: Configure the Inventory

1. **Copy the Sample Inventory**:
   - The sample inventory provides a template for defining the cluster nodes. Copying it allows you to create a customized configuration specific to your environment.
   ```bash
   cp -rfp inventory/sample inventory/kubecluster
   ```

2. **Define Node IPs**:
   - Define the IP addresses of the nodes to ensure that the Ansible playbooks target the correct machines.
   ```bash
   declare -a IPS=(10.255.0.51 10.255.0.52 10.255.0.53)
   CONFIG_FILE=inventory/kubecluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
   ```

3. **Edit `hosts.yaml`**:
   - Modify the `hosts.yaml` file to reflect your environment. This file defines the roles of each node and their IP addresses for Ansible to use during the deployment process.
   ```yaml
   all:
     hosts:
       knode01:
         ansible_host: 10.255.0.51
         ip: 10.255.0.51
         access_ip: 10.255.0.51
       knode02:
         ansible_host: 10.255.0.52
         ip: 10.255.0.52
         access_ip: 10.255.0.52
       knode03:
         ansible_host: 10.255.0.53
         ip: 10.255.0.53
         access_ip: 10.255.0.53
     children:
       kube_control_plane:
         hosts:
           knode01:
           knode02:
           knode03:
       kube_node:
         hosts:
           knode01:
           knode02:
           knode03:
       etcd:
         hosts:
           knode01:
           knode02:
           knode03:
       k8s_cluster:
         children:
           kube_control_plane:
           kube_node:
       calico_rr:
         hosts: {}
   ```

---

## Step 4: Deploy the Kubernetes Cluster

Deploy the cluster using the following command:

- This command runs the Ansible playbook to deploy the Kubernetes cluster. It sets up all necessary components, including the control plane, worker nodes, and networking.

```bash
ansible-playbook -i inventory/kubecluster/hosts.yaml --become --become-user=root cluster.yml
```

Verify the cluster status:

- Once the deployment is complete, use the following command to verify that all nodes are part of the cluster and in a `Ready` state:

```bash
kubectl get nodes
```

---

## Step 5: Reset the Cluster (Optional)

If you encounter issues during deployment, you can reset the cluster:

- This command resets the cluster to its initial state, allowing you to troubleshoot or re-deploy the cluster from scratch.

```bash
ansible-playbook -i inventory/kubecluster/hosts.yaml --become --become-user=root reset.yml
```

If the reset fails due to missing facts, delete the following directory and try again:

- Sometimes Ansible stores cached information that might interfere with the reset process. Deleting the cache resolves these issues.

```bash
rm -rf /root/.ansible
```

---

## Conclusion

This guide demonstrates how to deploy an HA Kubernetes cluster using Kubespray. The process simplifies Kubernetes setup and provides flexibility for configuration through Ansible playbooks.

For additional information, refer to the [Kubespray official documentation](https://github.com/kubernetes-sigs/kubespray).
