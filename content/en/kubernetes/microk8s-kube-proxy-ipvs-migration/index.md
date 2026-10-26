---
title: "Retiring kube-proxy IPVS on MicroK8s: An Upgrade Readiness Runbook"
description: "A controlled pre-upgrade runbook for MicroK8s clusters that still use kube-proxy IPVS mode: inventory, migrate to iptables, verify Service traffic, and retain a tested rollback path."
date: 2026-10-26
type: docs
author: Samuel Matildes
tags: [kubernetes, microk8s, kube-proxy, ipvs, iptables, networking, services, upgrades, runbook]
---

<i class="fas fa-route" aria-hidden="true"></i> A focused runbook for checking whether a MicroK8s cluster needs an IPVS migration before its next Kubernetes minor upgrade.

## Scope and outcome

This is not a general CNI replacement guide and it does not change Traefik, IngressRoutes, or application manifests. It covers one question: **does this cluster run kube-proxy in `ipvs` mode, and if so, can it move safely to `iptables` before the next MicroK8s minor upgrade?**

Kubernetes has deprecated kube-proxy IPVS mode. The upstream project expects it to be disabled by default by v1.40 and removed by v1.43. It also notes that IPVS mode still relies on iptables for part of Kubernetes Service behaviour, so it is not a complete replacement for iptables rules.[1]

If the audit finds `mode: iptables`, leave the proxy mode alone and record the clean result. A migration is only needed when the active kube-proxy configuration explicitly selects `ipvs`.

> **Change boundary:** run the inventory and verification commands first. Perform the mode switch only in an approved maintenance window, with a tested recovery decision. The examples use placeholders and must be adapted to the cluster under review.

## Why this change deserves a Service-focused test plan

Unless a cluster has deliberately installed an alternative, every node runs kube-proxy. It watches Services and EndpointSlices, then programs the node to capture a Service `clusterIP` and redirect traffic to an endpoint.[2]

That makes this a node-level Service-forwarding change. A successful Traefik dashboard or a healthy Deployment does not prove that `ClusterIP`, cross-namespace traffic, endpoint churn, or NodePorts still work after kube-proxy reconciles rules.

## 1. Capture the current state

Start with a read-only evidence bundle. Save command output with the maintenance record; do not copy private addresses, tokens, or real application hostnames into a public runbook.

```bash
# Run on the MicroK8s host.
sudo microk8s status --wait-ready
sudo snap list microk8s
sudo snap info microk8s

# Confirm the active kube-proxy mode.
sudo microk8s kubectl -n kube-system get configmap kube-proxy \
  -o jsonpath='{.data.config\.conf}' | grep -E '^mode:'

# Capture the full effective config for the change record.
sudo microk8s kubectl -n kube-system get configmap kube-proxy \
  -o yaml > kube-proxy-configmap.before.yaml

# Record kube-proxy placement, logs, Services, and EndpointSlices.
sudo microk8s kubectl -n kube-system get daemonset kube-proxy -o wide
sudo microk8s kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
sudo microk8s kubectl -n kube-system logs daemonset/kube-proxy --tail=200
sudo microk8s kubectl get svc -A -o wide
sudo microk8s kubectl get endpointslice -A
```

On an IPVS-based node, also capture the kernel-side evidence. These commands are diagnostic only; lack of `ipvsadm` does not prove the cluster is not using IPVS.

```bash
lsmod | grep '^ip_vs' || true
sudo ipvsadm -Ln 2>/dev/null || true
sudo iptables-save > iptables.before.txt
sudo nft list ruleset > nftables.before.txt 2>/dev/null || true
```

### Inventory the dependencies that change risk

Before touching kube-proxy, identify the resources whose traffic depends on it:

```bash
# Service types and their selectors.
sudo microk8s kubectl get svc -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP,PORTS:.spec.ports[*].port'

# Workloads and the CNI configuration currently in use.
sudo microk8s kubectl get deploy,daemonset,statefulset -A
sudo microk8s kubectl -n kube-system get configmap -o name
sudo microk8s kubectl get crd
```

Document separately:

- Services of type `ClusterIP`, `NodePort`, and `LoadBalancer`.
- Any MetalLB, external load balancer, or NodePort consumer.
- CNI customisations and NetworkPolicies.
- Ingress controllers and one representative route per controller.
- Stateful workloads, PVCs, and recovery requirements.

MicroK8s uses snap channels: patch releases stay within the installed channel, while a minor upgrade is an intentional refresh to another channel. Its documentation also warns that add-ons and the CNI are not upgraded automatically with the cluster and need independent release-note review.[3]

## 2. Establish recovery controls

Do not make the proxy mode change during a general application deployment. Use a maintenance window and decide in advance what restores service if verification fails.

Minimum controls:

1. Confirm recent workload/PVC backups and a restore test appropriate to the applications involved.
2. Save `kube-proxy-configmap.before.yaml`, current Service and EndpointSlice listings, and the node rule snapshots.
3. Record the current MicroK8s channel and whether a snap refresh is held.
4. Name an operator and a rollback decision point.
5. Define representative Service probes before the maintenance starts.

For a single-node cluster, a practical rollback decision is: if DNS, a same-namespace `ClusterIP`, a cross-namespace `ClusterIP`, or an externally used NodePort fails after the normal reconciliation window, restore the saved kube-proxy ConfigMap and restart the DaemonSet before investigating unrelated add-ons.

## 3. Prove the tests before the change

Create a temporary, disposable probe namespace. It gives the runbook a repeatable DNS and Service test without depending on an application-specific image.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: proxy-migration-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: proxy-migration-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo:1.0
          args: ["-text=proxy-ok"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: proxy-migration-test
spec:
  selector:
    app: echo
  ports:
    - name: http
      port: 80
      targetPort: 5678
```

Validate the illustrative manifest before applying it to any cluster:

```bash
sudo microk8s kubectl apply --dry-run=client -f proxy-migration-test.yaml
sudo microk8s kubectl apply --dry-run=server -f proxy-migration-test.yaml
```

After the approved apply, verify endpoints and in-cluster access:

```bash
sudo microk8s kubectl -n proxy-migration-test rollout status deploy/echo
sudo microk8s kubectl -n proxy-migration-test get svc,endpoints,endpointslice

sudo microk8s kubectl -n proxy-migration-test run dns-client \
  --rm -i --restart=Never --image=busybox:1.36 -- \
  sh -ec 'nslookup echo && wget -qO- http://echo'
```

For a cross-namespace check, run the same request from a short-lived Pod in an existing non-production namespace, targeting `http://echo.proxy-migration-test.svc.cluster.local`.

## 4. Change only the proxy mode

First inspect the stored `config.conf`. Do **not** replace the ConfigMap with a minimal hand-written one: it may contain cluster-specific settings that must remain intact.

```bash
sudo microk8s kubectl -n kube-system get configmap kube-proxy \
  -o jsonpath='{.data.config\.conf}' > kube-proxy.config.before.yaml
```

Edit a working copy and change only this field:

```yaml
mode: iptables
```

Then update the existing ConfigMap from the reviewed configuration. The exact key should be confirmed from the inventory command; the common key is `config.conf`.

```bash
# Replace only the existing config.conf value after peer review.
sudo microk8s kubectl -n kube-system create configmap kube-proxy \
  --from-file=config.conf=./kube-proxy.config.after.yaml \
  --dry-run=client -o yaml > kube-proxy-configmap.after.yaml

# Inspect the generated object before applying it.
sudo microk8s kubectl apply --dry-run=server -f kube-proxy-configmap.after.yaml
sudo microk8s kubectl apply -f kube-proxy-configmap.after.yaml

# Ask each kube-proxy Pod to reload from the updated configuration.
sudo microk8s kubectl -n kube-system rollout restart daemonset/kube-proxy
sudo microk8s kubectl -n kube-system rollout status daemonset/kube-proxy --timeout=5m
```

> **Important:** `kubectl create configmap ... -o yaml` produces an object definition. Before applying it, compare it with `kube-proxy-configmap.before.yaml` and preserve metadata and non-`config.conf` data required by the installed MicroK8s version. If that comparison is not clean, stop and use the platform's supported configuration workflow instead.

## 5. Verify in layers

Run every check below after kube-proxy is ready. Keep the output with the maintenance record.

### Control plane and proxy health

```bash
sudo microk8s kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
sudo microk8s kubectl -n kube-system logs daemonset/kube-proxy --tail=200
sudo microk8s kubectl -n kube-system get configmap kube-proxy \
  -o jsonpath='{.data.config\.conf}' | grep -E '^mode:'
```

Look for ready Pods, the expected `mode: iptables`, and no repeated proxy initialisation or rule-programming errors.

### Kubernetes DNS and ClusterIP traffic

```bash
# Same namespace: DNS plus Service forwarding.
sudo microk8s kubectl -n proxy-migration-test run same-ns \
  --rm -i --restart=Never --image=busybox:1.36 -- \
  sh -ec 'nslookup echo && wget -qO- http://echo'

# Cross namespace: fully-qualified service name.
sudo microk8s kubectl -n <probe-namespace> run cross-ns \
  --rm -i --restart=Never --image=busybox:1.36 -- \
  sh -ec 'nslookup echo.proxy-migration-test.svc.cluster.local && wget -qO- http://echo.proxy-migration-test.svc.cluster.local'
```

### Endpoint churn, NodePort, and ingress paths

- Scale the disposable `echo` Deployment from two replicas to one and back to two; confirm the EndpointSlice updates and requests keep succeeding.
- For every NodePort consumed outside the cluster, test it from the expected LAN client and record the result.
- Request one representative ingress hostname for every controller in use; verify the controller reaches its backing Service rather than only returning its own health page.
- Run each application's existing health check. A Service has no useful traffic path when it has no ready endpoints, regardless of kube-proxy mode.

Kubernetes documents that kube-proxy keeps node rules synchronized with Service and EndpointSlice state. That is why endpoint churn belongs in the verification set rather than being treated as an optional load test.[2]

## 6. Troubleshooting and rollback

| Symptom | Check first | Controlled response |
|---|---|---|
| kube-proxy Pods are not ready | DaemonSet events and Pod logs | Restore the saved ConfigMap if the mode change is implicated; do not edit the CNI first. |
| Service DNS works but requests fail | EndpointSlices, ready endpoints, and target ports | Confirm the Service selector and endpoint readiness before treating it as a proxy failure. |
| NodePort fails but ClusterIP works | Node firewall, NodePort listener, external client path | Compare node rules and firewall policy with the pre-change snapshots. |
| Intermittent traffic after the switch | kube-proxy logs, endpoint churn, stale node state | Wait for normal reconciliation once; if the symptom persists, roll back and collect node-level evidence. |
| Mode remains IPVS after an edit | Active ConfigMap key and DaemonSet arguments | Re-read the live ConfigMap; do not assume an edited local file was applied. |
| CNI or add-on failures appear during the maintenance | Component versions and release notes | Isolate the failure. MicroK8s add-ons and CNI updates are separate upgrade work, not a reason to widen this change.[3][4] |

### Rollback

```bash
# Restore the captured object only after confirming its namespace and contents.
sudo microk8s kubectl apply -f kube-proxy-configmap.before.yaml
sudo microk8s kubectl -n kube-system rollout restart daemonset/kube-proxy
sudo microk8s kubectl -n kube-system rollout status daemonset/kube-proxy --timeout=5m
```

Repeat the same DNS, ClusterIP, NodePort, ingress, and application probes. Record both the failed condition and the successful rollback result before scheduling another attempt.

## 7. Continue with the MicroK8s minor upgrade

Only after the proxy mode is recorded as `iptables` and the Service checks pass should the cluster move to a new MicroK8s minor channel. Review the target channel and the release notes, then use an approved change plan to refresh MicroK8s to that channel. MicroK8s explicitly separates the snap-channel upgrade from add-on and CNI lifecycle work, so assess those components before and after the refresh rather than assuming they update automatically.[3][4]

## Operator checklist

- [ ] Captured MicroK8s channel, version, kube-proxy ConfigMap, Services, EndpointSlices, and node rule evidence.
- [ ] Confirmed whether the active mode is `ipvs` or `iptables`.
- [ ] Stopped with **no migration required** when the cluster already used `iptables`.
- [ ] Reviewed workload recovery controls and rollback decision point.
- [ ] Validated the disposable probe manifest with both client and server dry-runs.
- [ ] Changed only kube-proxy configuration after review.
- [ ] Verified proxy readiness, DNS, same- and cross-namespace ClusterIP traffic, endpoint churn, NodePorts, ingress, and representative application health.
- [ ] Recorded the validated proxy mode before the MicroK8s minor upgrade.

## Sources

[1] https://kubernetes.io/blog/2026/07/31/kubernetes-v1-37-sneak-peek — Kubernetes v1.37 Sneak Peek
[2] https://kubernetes.io/docs/reference/networking/virtual-ips — Virtual IPs and Service Proxies
[3] https://canonical.com/microk8s/docs/upgrading — Upgrading MicroK8s
[4] https://canonical.com/microk8s/docs/release-notes — MicroK8s release notes
