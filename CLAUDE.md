# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbooks for provisioning a 3-node Kubernetes homelab cluster (1 control plane, 2 workers) with a full observability and AI agent stack.

## Commands

```bash
# Provision the base Kubernetes cluster (with password prompts)
cd kubernetes && ansible-playbook site.yml --ask-pass --ask-become-pass

# Deploy platform services (requires GEMINI_API_KEY for Kagent)
export GEMINI_API_KEY="your-key"
cd kubernetes && ansible-playbook services.yml --ask-pass --ask-become-pass

# If using SSH keys and passwordless sudo, omit --ask-pass --ask-become-pass

# Reset services only (keep base cluster intact)
cd kubernetes && ansible-playbook reset-services.yml --ask-pass --ask-become-pass

# Reset/decommission the entire cluster (removes Kubernetes and containerd completely)
cd kubernetes && ansible-playbook reset.yml --ask-pass --ask-become-pass

# Check Ansible syntax
ansible-playbook --syntax-check kubernetes/site.yml

# Dry run (check mode)
ansible-playbook --check kubernetes/site.yml

# Run specific role only
ansible-playbook kubernetes/services.yml --tags metallb

# Target specific hosts
ansible-playbook kubernetes/site.yml --limit control_plane
```

## Architecture

The setup uses two sequential playbooks:

1. **site.yml** - Cluster provisioning
   - `preflight` role validates node requirements (memory, CPU, disk, kernel, connectivity)
   - `common` role prepares all nodes (docker.io/containerd, kubeadm, kernel params)
   - `kubernetes` role initializes control plane then joins workers
   - Join token is passed via `/tmp/kubeadm_join_command` local file
   - Calico CNI installed via **Tigera operator** (not simple manifest)

2. **services.yml** - Platform services (runs on control plane only)
   - Deploys in order: Helm → MetalLB → Istio → Prometheus → Kiali → Kagent → kgateway → Ingress routes
   - All services integrate through Istio Gateway API for ingress
   - MetalLB provides LoadBalancer IPs from configured pool

3. **reset-services.yml** - Removes services only, keeps cluster
4. **reset.yml** - Full cluster teardown (removes K8s, containerd, kernel configs, repos)

## Key Configuration

- **group_vars/all.yml**: All cluster-wide variables (versions, CIDRs, hostnames, passwords)
- **inventory/hosts.yml**: Node IPs and SSH configuration
- Each role has `defaults/main.yml` for role-specific defaults

## Networking

- Pod CIDR: **10.10.0.0/16**
- Service CIDR: 10.96.0.0/12
- MetalLB pool: 192.168.1.24-29
- CNI: Calico via Tigera operator
- Service mesh: Istio ambient mode (sidecar-less)

## Component Versions (verified compatible)

| Component | Version | Notes |
|-----------|---------|-------|
| Kubernetes | 1.31 | Via kubeadm |
| containerd | via docker.io | Ubuntu package, not Docker repo |
| Calico | 3.28.0 | Tigera operator method |
| Istio | 1.27.1 | Supports K8s 1.29-1.33 |
| Gateway API | v1.4.0 | Required by Istio 1.27+ and kgateway |
| MetalLB | 0.14.9 | L2 mode |
| kube-prometheus-stack | 72.6.4 | Helm chart |
| Kiali | 2.20.0 | Operator install, requires 2.12+ for Istio 1.27 |
| kgateway | v2.1.2 | Requires Gateway API v1.4.0 |

## Important Implementation Details

- **Container runtime**: Uses `docker.io` package from Ubuntu repos (includes containerd), NOT `containerd.io` from Docker's repo. This avoids version conflicts.
- **Calico installation**: Uses Tigera operator + custom-resources.yaml, NOT the simple calico.yaml manifest. Pods run in `calico-system` namespace.
- **GPG keys**: Uses modern `/etc/apt/keyrings/` directory with `gpg --dearmor`, not deprecated `apt_key`.
- **Grafana password**: Must change from "changeme" in group_vars/all.yml before running services.yml.

## Troubleshooting Reference

- If containerd.io conflicts with docker.io: Run reset.yml to remove containerd.io and Docker repo
- If Calico pods crash: Check `/etc/cni/net.d/calico-kubeconfig` for malformed URLs
- If etcd won't start: Usually corrupted data from previous attempts - run reset.yml

## Maintenance

When making changes to this repository:
- Update README.md if adding new roles, changing architecture, or modifying configuration options
- Update this file (CLAUDE.md) if commands or architecture patterns change
- Verify version compatibility when updating any component (check upstream docs)
- **Commit and push changes** after completing modifications - don't leave uncommitted work

## Playbook Guidelines

**All playbook changes must be idempotent.** Running the playbook multiple times should produce the same result as running it once:
- Use `helm upgrade -i` instead of `helm install`
- Use `kubectl apply` instead of `kubectl create` (or handle "already exists" errors)
- Use `kubectl patch` for updating existing resources managed by Helm
- Check if resources/tokens exist before creating new ones
- Only restart deployments when configuration actually changes
- Use `changed_when` and `failed_when` conditions appropriately

## Reference Guide

Playbook aligned with: https://www.cherryservers.com/blog/install-kubernetes-ubuntu
