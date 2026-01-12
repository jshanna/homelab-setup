# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbooks for provisioning a 3-node Kubernetes homelab cluster (1 control plane, 2 workers) with a full observability and AI agent stack.

## Commands

```bash
# Provision the base Kubernetes cluster
cd kubernetes && ansible-playbook site.yml

# Deploy platform services (requires GEMINI_API_KEY for Kagent)
export GEMINI_API_KEY="your-key"
cd kubernetes && ansible-playbook services.yml

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
   - `common` role prepares all nodes (containerd, kubeadm, kernel params)
   - `kubernetes` role initializes control plane then joins workers
   - Join token is passed via `/tmp/kubeadm_join_command` local file

2. **services.yml** - Platform services (runs on control plane only)
   - Deploys in order: Helm → MetalLB → Istio → Prometheus → Kiali → Kagent → kgateway → Ingress routes
   - All services integrate through Istio Gateway API for ingress
   - MetalLB provides LoadBalancer IPs from configured pool

## Key Configuration

- **group_vars/all.yml**: All cluster-wide variables (versions, CIDRs, hostnames, passwords)
- **inventory/hosts.yml**: Node IPs and SSH configuration
- Each role has `defaults/main.yml` for role-specific defaults

## Networking

- Pod CIDR: 10.244.0.0/16
- Service CIDR: 10.96.0.0/12
- MetalLB pool: 192.168.1.24-29
- CNI: Calico
- Service mesh: Istio ambient mode (sidecar-less)

## Maintenance

When making changes to this repository, keep documentation in sync:
- Update README.md if adding new roles, changing architecture, or modifying configuration options
- Update this file (CLAUDE.md) if commands or architecture patterns change
