# Homelab Kubernetes Setup

Ansible playbooks for provisioning a production-like Kubernetes cluster on homelab infrastructure running Ubuntu Server 24.04 LTS. This setup deploys a 3-node cluster with a full observability stack, service mesh, and AI agent infrastructure.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Homelab Network                          │
│                       192.168.1.0/24                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   homelab1   │  │   homelab2   │  │   homelab3   │          │
│  │ Control Plane│  │    Worker    │  │    Worker    │          │
│  │ 192.168.1.21 │  │ 192.168.1.22 │  │ 192.168.1.23 │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              MetalLB IP Pool: 192.168.1.24-29           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Version | Description |
|-----------|---------|-------------|
| Kubernetes | 1.31 | Container orchestration platform |
| containerd | (via docker.io) | Container runtime |
| Calico | 3.28.0 | CNI plugin via Tigera operator |
| MetalLB | 0.14.9 | Bare-metal load balancer |
| Istio | 1.27.1 | Service mesh (ambient mode) |
| Prometheus | kube-prometheus-stack 72.6.4 | Metrics collection and alerting |
| Grafana | (via kube-prometheus-stack) | Metrics visualization |
| InfluxDB | 3 Core | Time-series database (official image) |
| InfluxDB Explorer | latest | InfluxDB 3 web UI |
| MongoDB | latest | Document database (Bitnami chart) |
| Mongo Express | 1.0.2 | MongoDB web UI |
| Kiali | 2.20.0 | Istio service mesh observability |
| Kagent | latest | Kubernetes AI agent |
| kgateway | v2.1.2 | Agent gateway for AI workloads |

## Prerequisites

- 3 Ubuntu Server 24.04 LTS nodes with a sudo-enabled user
- Ansible installed on your control machine
- Network connectivity between all nodes
- (Optional) GEMINI_API_KEY for Kagent

### Node Setup

Create an `ansible` user on each node with sudo privileges:

```bash
# On each node (as root)
useradd -m -s /bin/bash ansible
echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
passwd ansible  # Set a password for SSH access

# Or for password-based sudo (more secure):
echo "ansible ALL=(ALL) ALL" > /etc/sudoers.d/ansible
```

For passwordless operation, copy your SSH key:
```bash
ssh-copy-id ansible@192.168.1.21
ssh-copy-id ansible@192.168.1.22
ssh-copy-id ansible@192.168.1.23
```

## Quick Start

### 1. Clone the repository

```bash
git clone <repository-url>
cd homelab-setup/kubernetes
```

### 2. Configure inventory

Edit `inventory/hosts.yml` to match your node IPs:

```yaml
all:
  children:
    k8s_cluster:
      children:
        control_plane:
          hosts:
            homelab1:
              ansible_host: 192.168.1.21
        workers:
          hosts:
            homelab2:
              ansible_host: 192.168.1.22
            homelab3:
              ansible_host: 192.168.1.23
```

### 3. Configure cluster settings

Edit `group_vars/all.yml` to customize:

- `metallb_ip_range`: Available IPs for LoadBalancer services
- `grafana_admin_password`: Grafana admin password (change from default!)
- `*_hostname`: DNS hostnames for services

### 4. Provision the cluster

```bash
cd kubernetes

# With password prompts (SSH + sudo):
ansible-playbook site.yml --ask-pass --ask-become-pass

# Or if using SSH keys and passwordless sudo:
ansible-playbook site.yml
```

### 5. Deploy services

```bash
export GEMINI_API_KEY="your-api-key"  # Required for Kagent

# With password prompts:
ansible-playbook services.yml --ask-pass --ask-become-pass

# Or if using SSH keys and passwordless sudo:
ansible-playbook services.yml
```

## Directory Structure

```
kubernetes/
├── ansible.cfg              # Ansible configuration
├── site.yml                 # Main playbook - cluster provisioning
├── services.yml             # Services playbook - addon deployment
├── reset.yml                # Decommission/reset the entire cluster
├── reset-services.yml       # Remove services only (keep cluster)
├── inventory/
│   └── hosts.yml            # Node inventory
├── group_vars/
│   └── all.yml              # Global variables
├── roles/
│   ├── preflight/           # Pre-installation validation checks
│   ├── common/              # Base system setup (containerd, kubeadm)
│   ├── kubernetes/          # Cluster initialization and node join
│   ├── metallb/             # MetalLB load balancer
│   ├── istio/               # Istio service mesh (ambient mode)
│   ├── prometheus/          # kube-prometheus-stack
│   ├── influxdb/            # InfluxDB 2.x time-series database
│   ├── mongodb/             # MongoDB with Mongo Express UI
│   ├── kiali/               # Kiali service mesh dashboard
│   ├── kagent/              # Kubernetes AI agent
│   ├── kgateway/            # Agent gateway
│   └── ingress/             # HTTP route configuration
└── reference/
    └── service_links.txt    # Links to upstream projects
```

## Playbooks

### site.yml

Provisions the base Kubernetes cluster:

1. Runs preflight checks (memory, CPU, disk, kernel, network connectivity)
2. Prepares all nodes (disables swap, installs containerd, kubeadm)
3. Initializes the control plane with Calico CNI
4. Joins worker nodes to the cluster
5. Installs local-path-provisioner for persistent storage

### services.yml

Deploys platform services on top of the cluster:

1. Installs Helm
2. Deploys MetalLB for LoadBalancer support
3. Deploys Istio in ambient mode
4. Deploys kube-prometheus-stack (Prometheus + Grafana)
5. Deploys InfluxDB 2.x time-series database
6. Deploys MongoDB with Mongo Express web UI
7. Deploys Kiali for service mesh visualization
8. Deploys Kagent (AI agent for Kubernetes)
9. Deploys kgateway (AgentGateway)
10. Configures ingress routes via Istio Gateway

### reset-services.yml

Removes all services deployed by services.yml while keeping the base cluster intact:

1. Removes ingress routes (HTTPRoutes)
2. Removes InfluxDB and MongoDB (including Mongo Express)
3. Removes Kagent and kgateway
4. Removes Kiali operator and CR
5. Removes kube-prometheus-stack and CRDs
6. Uninstalls Istio and Gateway API CRDs
7. Removes MetalLB
8. Cleans up Helm repositories

```bash
ansible-playbook reset-services.yml --ask-pass --ask-become-pass
```

### reset.yml

Completely decommissions the cluster, removing everything installed by site.yml:

1. Stops and resets kubeadm
2. Removes all containers and pods
3. Removes Kubernetes and containerd packages
4. Cleans up all directories and state
5. Removes apt repositories and GPG keys
6. Removes kernel module and sysctl configurations
7. Cleans up iptables and network interfaces

```bash
ansible-playbook reset.yml --ask-pass --ask-become-pass
```

## Accessing Services

After deployment, configure DNS entries pointing to the gateway IP:

| Service | URL | Default Port |
|---------|-----|--------------|
| Grafana | http://grafana.homelab.local | 80 |
| Prometheus | http://prometheus.homelab.local | 80 |
| Alertmanager | http://alertmanager.homelab.local | 80 |
| InfluxDB API | http://influxdb.homelab.local | 80 |
| InfluxDB Explorer | http://influxdb-explorer.homelab.local | 80 |
| MongoDB (Mongo Express) | http://mongodb.homelab.local | 80 |
| Kiali | http://kiali.homelab.local | 80 |
| Kagent | http://kagent.homelab.local | 80 |
| AgentGateway | http://agentgateway.homelab.local/ui | 80 |

The gateway IP is assigned by MetalLB from the configured pool (192.168.1.24-29).

### Local DNS Setup

Add entries to your local DNS server or `/etc/hosts`:

```
192.168.1.24  grafana.homelab.local
192.168.1.24  prometheus.homelab.local
192.168.1.24  alertmanager.homelab.local
192.168.1.24  influxdb.homelab.local
192.168.1.24  influxdb-explorer.homelab.local
192.168.1.24  mongodb.homelab.local
192.168.1.24  kiali.homelab.local
192.168.1.24  kagent.homelab.local
192.168.1.24  agentgateway.homelab.local
```

## Configuration Reference

### group_vars/all.yml

| Variable | Default | Description |
|----------|---------|-------------|
| `kubernetes_version` | 1.31 | Kubernetes minor version |
| `pod_network_cidr` | 10.10.0.0/16 | Pod network CIDR |
| `service_cidr` | 10.96.0.0/12 | Service network CIDR |
| `cni_plugin` | calico | CNI plugin (calico) |
| `metallb_ip_range` | 192.168.1.24-192.168.1.29 | MetalLB IP pool |
| `istio_version` | 1.27.1 | Istio version |
| `istio_profile` | ambient | Istio installation profile |
| `grafana_admin_password` | prom-operator | Grafana admin password (do NOT use "admin") |
| `prometheus_retention` | 15d | Prometheus data retention |
| `prometheus_storage_size` | 50Gi | Prometheus storage size |
| `influxdb_admin_token` | changeme | InfluxDB bearer token (MUST change!) |
| `influxdb_storage_size` | 50Gi | InfluxDB storage size |
| `influxdb_explorer_session_secret` | changeme | Explorer session key (MUST change!) |
| `mongodb_root_password` | changeme | MongoDB root password (MUST change!) |
| `mongodb_architecture` | standalone | MongoDB architecture (standalone/replicaset) |
| `mongodb_storage_size` | 20Gi | MongoDB storage size |
| `kiali_version` | 2.20.0 | Kiali version (2.12+ required for Istio 1.27) |
| `kagent_provider` | gemini | AI provider for Kagent |

## Roles

### preflight

Validates node requirements before installation:
- Checks minimum memory (2GB), CPU cores (2), and disk space (20GB)
- Verifies kernel version compatibility
- Tests network connectivity between nodes
- Warns if swap is enabled

### common

Prepares nodes for Kubernetes:
- Installs required packages
- Disables swap
- Configures kernel modules (overlay, br_netfilter)
- Sets sysctl parameters for networking
- Installs docker.io (includes containerd)
- Configures containerd with systemd cgroup
- Installs kubeadm, kubelet, kubectl

### kubernetes

Initializes cluster and joins nodes:
- **Control plane**: Runs `kubeadm init`, installs Calico via Tigera operator, sets up local-path-provisioner
- **Workers**: Joins nodes using token from control plane

### metallb

Deploys MetalLB bare-metal load balancer:
- Installs via Helm
- Configures IP address pool
- Sets up L2 advertisement

### istio

Deploys Istio service mesh in ambient mode:
- Installs istioctl
- Installs Gateway API CRDs
- Deploys Istio with ambient profile
- Creates ingress gateway
- Labels namespaces for ambient mode (excludes kube-system, istio-system, kgateway-system)

### prometheus

Deploys kube-prometheus-stack:
- Prometheus for metrics collection
- Grafana for visualization
- Alertmanager for alerts
- Configures persistent storage

### influxdb

Deploys InfluxDB 3 Core time-series database:
- Uses official `influxdb:3-core` Docker image (not Helm)
- Direct Kubernetes deployment for reliability
- Uses bearer token authentication
- ServiceMonitor for Prometheus metrics
- Persistent storage with configurable size
- Handles migration from older deployments automatically
- Deploys InfluxDB 3 Explorer web UI for interactive SQL queries

### mongodb

Deploys MongoDB document database:
- Installs via Bitnami Helm chart
- Configures authentication and persistence
- Deploys Mongo Express web UI for database management
- Enables metrics exporter with automatic ServiceMonitor

### kiali

Deploys Kiali service mesh dashboard:
- Installs Kiali operator
- Configures token-based authentication (more secure than anonymous)
- Configures integration with Prometheus and Grafana

To login to Kiali, generate a token:
```bash
kubectl -n istio-system create token kiali-service-account
```

### kagent

Deploys Kagent Kubernetes AI agent:
- Requires GEMINI_API_KEY environment variable
- Installs CRDs and controller
- Deploys UI component
- Automatically configures Grafana MCP integration (creates service account and API token)

### kgateway

Deploys kgateway (AgentGateway):
- Installs CRDs
- Deploys controller with agentgateway enabled
- Configures admin UI to bind to all interfaces (accessible via /ui endpoint)
- Creates GatewayParameters for custom configuration

### ingress

Configures HTTP routes via Istio Gateway API:
- Creates HTTPRoute resources for all services
- Routes traffic based on hostname

## Troubleshooting

### Check cluster status

```bash
kubectl get nodes
kubectl get pods -A
```

### Check service mesh

```bash
istioctl analyze
kubectl get gateway -A
```

### View logs

```bash
kubectl logs -n istio-system -l app=istiod
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus
```

### Reset services or cluster

To remove only services (keep base cluster):
```bash
ansible-playbook reset-services.yml --ask-pass --ask-become-pass
```

To completely decommission the cluster:
```bash
ansible-playbook reset.yml --ask-pass --ask-become-pass
```

See the [Playbooks](#playbooks) section for details on what each reset playbook removes.

## References

- [Istio](https://github.com/istio/istio)
- [Prometheus](https://github.com/prometheus/prometheus)
- [Grafana](https://github.com/grafana/grafana)
- [InfluxDB](https://github.com/influxdata/influxdb)
- [MongoDB](https://github.com/mongodb/mongo)
- [Mongo Express](https://github.com/mongo-express/mongo-express)
- [Kiali](https://github.com/kiali/kiali)
- [Kagent](https://github.com/kagent-dev/kagent)
- [AgentGateway](https://github.com/agentgateway/agentgateway)
