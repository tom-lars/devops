# Istio Upgrade Guide

This guide documents the process for upgrading Istio from version 1.6.5 to version 1.26.0 in a Kubernetes development environment.

## Overview

This upgrade will:
- Update the Istio control plane to the latest version compatible with Kubernetes
- Update all Envoy sidecar proxies
- Maintain zero downtime during the upgrade
- Ensure service mesh functionality post-upgrade

## Environment Information

| Component | Version |
|-----------|---------|
| Kubernetes | v1.31.6-gke.1064001 |
| Istio (Before) | 1.6.5 |
| Istio (After) | 1.26.0 |
| Installation Method | istioctl |
| Affected Namespaces | default, istio-system |

## Prerequisites

Before beginning the upgrade process, ensure you have:

- Access to the Kubernetes cluster
- Administrative permissions
- kubectl installed and configured
- Backup storage location available

## Pre-Upgrade Checklist

### 1. Verify Current Versions

Check the current Istio version:
```bash
# If istioctl is available
istioctl version

# Alternative: Check control plane version
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=istiod -o jsonpath='{.items[0].metadata.name}') -- /usr/local/bin/pilot-discovery version

# Check via deployment image tags
kubectl get deployment istiod -n istio-system -o yaml | grep image:
```

Check Kubernetes version:
```bash
kubectl version
```

### 2. Verify Istio Installation Status

Check if Istio CRDs are installed:
```bash
kubectl get crds | grep 'istio.io'
```

Verify the istio-system namespace:
```bash
kubectl get ns istio-system
kubectl get pod -n istio-system
```

Check for Istio control plane components:
```bash
kubectl get pods -n istio-system
```

Verify webhook configurations:
```bash
kubectl get mutatingwebhookconfigurations | grep istio
kubectl get validatingwebhookconfigurations | grep istio
```

### 3. Verify DNS and Connectivity

Check DNS resolution:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### 4. Verify Sidecar Injection Status

Check if auto-injection is enabled:
```bash
kubectl get namespace default --show-labels
```

## Backup Procedure

Before proceeding with the upgrade, create a backup of your Istio configuration:

```bash
# Backup all Istio resources
kubectl get all -n istio-system -o yaml > /backups/istio/istio-backup.yaml

# Backup Istio CRDs
kubectl get crds | grep istio
kubectl get crds -o yaml > /backups/istio/crds-backup.yaml

# Backup service mesh custom resources
kubectl get virtualservices,destinationrules,gateways,envoyfilters,authorizationpolicies,peerauthentications -A -o yaml > /backups/istio/istio-crs.yaml
```

## Upgrade Process

We'll be using the **Canary Upgrade** approach as it provides a safe and gradual rollout with easy rollback options.

### 1. Download Target Istio Version

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.26.0 TARGET_ARCH=x86_64 sh -
cd istio-1.26.0
export PATH=$PWD/bin:$PATH
```

### 2. Add Egress Gateway (if not present)

```bash
istioctl install --set components.egressGateways[0].name=istio-egressgateway --set components.egressGateways[0].enabled=true

# Update the egress gateway
istioctl upgrade \
  --set components.ingressGateways[0].enabled=true \
  --set components.ingressGateways[0].name=istio-ingressgateway \
  --set components.egressGateways[0].enabled=true \
  --set components.egressGateways[0].name=istio-egressgateway

# Verify the egress gateway
kubectl get pod -l istio=egressgateway -n istio-system
```

### 3. Install New Istio Version with Revision Tag

```bash
istioctl install \
  --set revision=1-26-0 \
  --set profile=default \
  --set components.ingressGateways[0].enabled=true \
  --set components.ingressGateways[0].name=istio-ingressgateway \
  --set components.egressGateways[0].enabled=true \
  --set components.egressGateways[0].name=istio-egressgateway \
  -y
```

### 4. Update Namespace Labels for New Revision

```bash
kubectl label namespace default istio.io/rev=canary-1-26-0 --overwrite
```

### 5. Enable Automatic Sidecar Injection

```bash
# For each namespace requiring sidecar injection
kubectl label namespace <namespace> istio-injection=enabled --overwrite
```

### 6. Restart Workloads

```bash
kubectl rollout restart deployment -n <namespace>
```

## Validation Steps

After the upgrade, perform these validation checks:

### 1. Confirm New Istio Version

```bash
istioctl version
```

### 2. Verify Sidecar Injection Status

```bash
kubectl get namespace default --show-labels
kubectl describe pod <podname>
```

### 3. Check Proxy Status and Synchronization

```bash
istioctl proxy-status
```

### 4. Analyze Mesh Configuration

```bash
istioctl analyze
```

### 5. Validate Ingress Gateway Traffic

```bash
curl -I http://<external-ip-or-domain>
```

### 6. Check mTLS Status (if applicable)

```bash
istioctl authn tls-check <pod-name>.<namespace>
```

## Component Verification

### Control Plane (istiod)

```bash
# Get istiod pods
kubectl get pods -n istio-system -l app=istiod

# Check logs
kubectl logs -n istio-system <istiod-pod-name>
```

### Ingress Gateway

```bash
# Get pods
kubectl get pods -n istio-system -l istio=ingressgateway

# Check logs
kubectl logs -n istio-system <ingress-pod-name>

# Check external IP
kubectl get svc istio-ingressgateway -n istio-system

# Test functionality
curl -I http://<external-ip>
```

### Egress Gateway

```bash
kubectl get pods -n istio-system -l istio=egressgateway
```

### API Version Check

```bash
kubectl get crds | grep 'istio.io'
```

## Verification Summary

| Check | Command | Expected Result |
|-------|---------|----------------|
| Istio version | `istioctl version` | All components show upgraded version |
| Sidecar injection | `kubectl get pod` | All pods have istio-proxy |
| Proxy sync | `istioctl proxy-status` | All proxies SYNCED |
| Config analysis | `istioctl analyze` | No validation errors |
| Service test | `curl between services` | 200 OK |
| Ingress test | `curl external` | Returns expected status |
| mTLS status | `istioctl authn tls-check` | mTLS working (if enforced) |

## Verify Component Images Are Updated

### Istiod (Control Plane)

```bash
kubectl get pods -n istio-system
kubectl get deployment istiod-1-26-0 -n istio-system -o=jsonpath='{.spec.template.spec.containers[*].image}'
```

### Ingress Gateway

```bash
kubectl get pods -n istio-system -l istio=ingressgateway -o wide
kubectl get deployment istio-ingressgateway -n istio-system -o=jsonpath='{.spec.template.spec.containers[*].image}'
```

### Egress Gateway

```bash
kubectl get pods -n istio-system -l istio=egressgateway -o wide
kubectl get deployment istio-egressgateway -n istio-system -o=jsonpath='{.spec.template.spec.containers[*].image}'
```

## Rollback Procedure (If Needed)

If issues are encountered during the upgrade process:

### 1. Remove New Revision Label

```bash
kubectl label namespace default istio.io/rev- --overwrite
```

### 2. Restart Deployments

```bash
kubectl rollout restart deployment -n default
```

### 3. Verify Rollback Success

```bash
istioctl proxy-status
```

## Cleanup Previous Version

Once the upgrade is verified successful and stable:

### 1. Verify All Namespaces Use New Revision

```bash
kubectl get namespace --show-labels | grep istio.io/rev
```

### 2. Check No Pods Use Old Sidecars

```bash
istioctl proxy-status
```

### 3. Delete Old Control Plane Components

```bash
kubectl get deployments -n istio-system
kubectl delete deployment istiod-old-version -n istio-system
```

### 4. Clean Up Old Webhooks

```bash
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Delete if referencing old revision
kubectl delete mutatingwebhookconfiguration <name>
kubectl delete validatingwebhookconfiguration <name>
```

## Upgrade Methods Comparison

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| Canary Upgrade | Safe and gradual rollout<br>Supports parallel testing<br>Easy rollback via label change | Slightly more complex<br>Requires namespace labeling and pod restarts | Recommended for dev and prod with minimal downtime |
| In-Place Upgrade | Fastest and simplest<br>One-line upgrade command | Risky (no rollback)<br>All workloads upgraded at once<br>Potential downtime | Small clusters or patch version bumps |
| Upgrade with Helm | Declarative and GitOps-friendly<br>Easily integrated with CI/CD<br>Supports rollback via helm | Adds Helm complexity<br>Requires Helm-based installation<br>More upfront configuration needed | Teams already using Helm and GitOps workflows |

## Why Canary Upgrade Is Recommended

For upgrading from Istio 1.6.5 to 1.26.0 (a major version jump), the Canary Upgrade approach is recommended because:

- It allows testing the new version without affecting running workloads
- Rollback is fast (just remove the revision label and restart pods)
- It minimizes downtime during the upgrade
- It provides a safer path for such a significant version jump

## Troubleshooting Tips

- If you encounter issues with webhook configurations, check for conflicts between old and new versions
- For sidecar injection issues, verify the namespace labels and restart the affected pods
- If proxy synchronization issues occur, check connectivity between Istiod and the proxies
- For traffic routing problems, verify the Ingress and Egress gateway configurations

## Additional Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio Upgrade Guide](https://istio.io/latest/docs/setup/upgrade/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
