# AWX Infrastructure Automation for VMs

This repository contains the configuration and automation manifests to deploy **AWX** via **K3s** to automate software version tracking and configuration management across a distributed fleet of **Zammad** helpdesk instances.

---

## ❓ Why AWX-Operator for Zammad?

Managing multiple Zammad installations across various VMs requires consistent visibility. We use the **AWX-Operator** to deploy a scalable AWX instance that:
1.  **Automates Inventory Discovery:** Automatically fetches and updates the list of VMs where Zammad is hosted.
2.  **Software Version Tracking:** Runs scheduled Ansible playbooks to query installed versions of Zammad, Elasticsearch, PostgreSQL, and Redis across all nodes.
3.  **Centralized Reporting:** Aggregates system health and version data into a single dashboard, replacing manual version checking.

---

## 📋 Prerequisites

- **OS:** Ubuntu 22.04 LTS or compatible Linux distribution.
- **Hardware:** Minimum 2 CPUs / 4GB RAM.
- **Tools:** `curl`, `git`, `kubectl`, and `kustomize`.

---

## 🚀 Installation

### 1. Install K3s (Lightweight Kubernetes)
K3s provides the lightweight orchestration layer needed to run the AWX-Operator.

```bash
# Install K3s
curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -s - --write-kubeconfig-mode 644

# Export the config to environment
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Verify the cluster is running
kubectl get nodes
```

# Deploy AWX Operator

The operator automates the complex setup of AWX (database migrations, task runners, and web services).

## Create the namespace
```bash
kubectl create namespace awx
```

## Setup installation directory
``` bash
mkdir -p ~/awx-install && cd ~/awx-install
```

## Create kustomization.yaml
``` yml
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1

images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1

namespace: awx
EOF
```
## Apply the operator deployment
``` bash
kubectl apply -k .
```

# Deploying the AWX Instance

Apply the AWX Custom Resource to trigger the deployment of the web and task engine pods.
1. Create awx-instance.yaml
```yaml
apiVersion: [awx.ansible.com/v1beta1](https://awx.ansible.com/v1beta1)
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
  projects_persistence: true
  projects_storage_class: local-path

```
2. Apply and monitor

```bash
kubectl apply -f awx-instance.yaml -n awx

# Monitor until all pods (web, task, postgres) are 'Running'
kubectl get pods -n awx -w
```

# 🔑 Access & Credentials

Once the awx-web and awx-task pods are ready:

1. Retrieve Admin Password:
```bash
kubectl get secret awx-admin-password -n awx -o jsonpath='{.data.password}' | base64 --decode ; echo

```

2. Access the UI:
Open your browser to: http://<YOUR_VM_IP>:30080

    Username: admin

    Password: (Output from the command above)


## 📂 Project Structure

```text
.
├── collections/                # Ansible collections (community.general, etc.)
├── group_vars/                 # Variable definitions for Zammad host groups
├── inventory/                  # Host definitions for the Zammad fleet
├── roles/
│   └── zammad_inventory/       # Templates and logic for version extraction
├── check_collections.yml       # Environment verification playbook
├── grafana_push.yml            # Metrics and data export to Grafana
└── host_reachability_playbook.yml # Network and SSH heartbeat check
```

### Key Playbooks
* **Version Discovery:** Uses `roles/zammad_inventory` to fetch installed software versions from many VMs.
* **Monitoring Integration:** `grafana_push.yml` automates the visualization of your Zammad VMs status to grafana server.