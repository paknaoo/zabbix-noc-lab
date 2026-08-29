# Troubleshooting: UFW Risk Assessment on Kubernetes Nodes

**Phase:** 05 — Zabbix Agent2 Deployment Across Monitored Hosts

## Context

Before adding a UFW rule to permit Zabbix agent2 polling on port 10050, `ss -tulnp` was run on `k8s-worker1` to check what the host already exposed on the network. This revealed several critical, cluster-wide services listening on `0.0.0.0` (all interfaces): kubelet (`10250`), Cilium VXLAN (`8472/UDP`), and Cilium health checks (`4240`/`4244`).

## Incident

While reviewing the same information on `k8s-master`, `sudo ufw enable` was run **before the full consequences had been confirmed**. UFW had previously been `inactive` on this host. Once enabled, with only SSH and port 10050 permitted, the control-plane node — which additionally serves `kube-apiserver` (`6443`), `etcd` (`2379`/`2380`), `kube-scheduler` and `kube-controller-manager` — was left with `deny incoming` applied to every other cluster port, for a period of several minutes.

## Risk Assessment

During this window, `kubectl get nodes` and `kubectl get pods -A` were checked and showed all nodes `Ready` and all pods `Running`. **No failure was observed.** This is not the same as confirming the change was safe: the most likely explanation is that UFW and conntrack do not retroactively terminate already-established connections, and no cluster component happened to restart or attempt a new connection during that window. Had any component restarted — kubelet, `cilium-agent`, or `etcd` re-establishing a peer connection — a new connection attempt on a now-blocked port could plausibly have failed. The absence of a visible symptom in a short window is not evidence of safety; it reflects the specific timing of what did and did not attempt to reconnect during that window.

## Corrective Action

`sudo ufw disable` was run on `k8s-master` immediately upon recognising this risk. `kubectl get nodes` and `kubectl get pods -A` were re-checked afterwards and confirmed the cluster remained fully healthy throughout.

## Final Decision

**UFW remains disabled on `k8s-master`, `k8s-worker1` and `k8s-worker2`.** Access to Zabbix agent2's port 10050 on these three nodes is secured entirely through a pfSense rule (`HOST_ZABBIX → GRP_K8S_NODES : 10050`), rather than through a second, host-level layer.

**Rationale:** these nodes belong to a separate project (`k8s-cilium-lab`) with its own, already-established security architecture — Cilium NetworkPolicies, confirmed present via the `netpol-demo` and `deny-demo` namespaces already in the cluster. Reconstructing a complete, correct UFW rule set covering every critical cluster service (kubelet, etcd, the API server, Cilium's VXLAN and health-check ports) on a host that belongs to another project's production-like environment carries disproportionate risk relative to the benefit of an additional defence-in-depth layer for a single Zabbix port that is already restricted at the network level via pfSense.

`mgmt`, by contrast, sits outside the Kubernetes cluster in the OUTSIDE network, so enabling UFW there carries no equivalent risk — see [Phase 05](../phase-05-agent2-nodes.md#agent2-installation-and-firewall-mgmt) for its configuration.

## Portfolio Note

This incident is a useful demonstration of three things:

1. **Checking before changing.** `ss -tulnp` surfaced the presence of critical, unfamiliar cluster services on the host before a firewall policy was applied — good practice on any host whose full service inventory isn't already known from memory.
2. **Recognising and reversing a risky change quickly**, before it caused an actual failure — treating "no visible symptom" as a warning sign to investigate further, not as confirmation of safety.
3. **A deliberate boundary decision between two projects.** Rather than uniformly applying the same host-hardening policy everywhere, the decision to leave UFW disabled on the Kubernetes nodes was made explicitly, with the reasoning about scope and risk recorded here — not left as an unexplained inconsistency for a future reader to puzzle over.
