        
# Node kernel prerequisites

Prepares the operating system on the cluster nodes (cp1, worker1, worker2) for Kubernetes:
swap off, required kernel modules loaded, and networking settings enabled.
All changes apply immediately and survive a reboot.

- **Playbook:** `ansible/playbooks/cluster-nodes/kernel-prereqs.yml`
- **Targets:** inventory group `nodes`
- **Run from:** the repo root on the control machine

## What it sets

| Area | Setting | Applied now by | Persisted in |
|---|---|---|---|
| Swap | zram swap off | `swapoff -a` (only if swap is active) | `/etc/systemd/zram-generator.conf` (empty file) |
| Kernel modules | `overlay` | `community.general.modprobe` | `/etc/modules-load.d/overlay.conf` |
| Kernel modules | `br_netfilter` | `community.general.modprobe` | `/etc/modules-load.d/br_netfilter.conf` |
| Networking | `net.ipv4.ip_forward = 1` | `ansible.posix.sysctl` | `/etc/sysctl.d/kubernetes.conf` |
| Networking | `net.bridge.bridge-nf-call-iptables = 1` | `ansible.posix.sysctl` | `/etc/sysctl.d/kubernetes.conf` |
| Networking | `net.bridge.bridge-nf-call-ip6tables = 1` | `ansible.posix.sysctl` | `/etc/sysctl.d/kubernetes.conf` |

### Why each one is needed

- **Swap off:** the kubelet expects swap to be off by default.
- **`overlay`:** lets CRI-O stack container image layers.
- **`br_netfilter`:** makes traffic crossing a network bridge visible to iptables. It is also what
  makes the two `net.bridge.*` settings exist.
- **`ip_forward`:** lets the node pass traffic between networks (pods ↔ other nodes).
- **`bridge-nf-call-iptables` / `ip6tables`:** sends bridged pod traffic through iptables so
  Kubernetes Services and Calico rules apply to it.

### Notes

- Fedora's zram swap is created at boot by `zram-generator` from
  `/usr/lib/systemd/zram-generator.conf`. It is **not** listed in `/etc/fstab`, so there is no
  fstab line to remove. An empty file with the same name in `/etc/systemd/` overrides the default
  and stops the swap device being created.
- The modules must be loaded **before** the sysctl settings are applied, because the
  `net.bridge.*` settings only exist once `br_netfilter` is loaded. The playbook runs the tasks
  in that order, and at boot systemd loads modules before it applies sysctl files.
- No firewalld on the nodes. We're using hypervisors firewalld rules

## Prerequisites

Install the collections the repo needs

```bash
ansible-galaxy collection install -r ansible/collections/requirements.yml
```

## Apply

```bash
ansible-playbook ansible/playbooks/cluster-nodes/kernel-prereqs.yml
```

A second run should report `changed=0` on every node.

## Verify

```bash
# Swap: expect empty output on every node
ansible nodes -a "swapon --show"

# Modules: expect overlay and br_netfilter (bridge also appears, loaded as a dependency)
ansible nodes -m shell -a "lsmod | grep -E 'overlay|br_netfilter'"

# Networking: expect all three = 1
ansible nodes -a "sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables"
```

To confirm the settings persist, reboot the nodes and repeat the checks:

```bash
ansible nodes -m ansible.builtin.reboot
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `couldn't resolve module/action 'community.general.modprobe'` | Collection not installed, or the editor can't see it | Run the prerequisites step; check with `ansible-doc community.general.modprobe`; reload the editor |
| `sysctl: cannot stat /proc/sys/net/bridge/...` | `br_netfilter` not loaded | Check `lsmod`; re-run the playbook; check `/etc/modules-load.d/br_netfilter.conf` exists |
| Swap is back after a reboot | zram override file missing or not empty | Check `/etc/systemd/zram-generator.conf` exists and is empty; re-run the playbook |