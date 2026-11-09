# How the VMs reach the internet and home network

- **Status:** Accepted
- **Date:** 2026-11-09
- **Ticket:** #7 Configure firewalld forwarding and selective NAT

## The problem

- My two mini PCs (kvm1 `192.168.18.201` and kvm2 `192.168.18.202`) connect to my home network over Wi-Fi only. Because of that, the VMs on each mini PC live on their own private network:
  - kvm1's VMs: `10.10.1.0/24`
  - kvm2's VMs: `10.10.2.0/24`
- Each mini PC already knows how to reach the other one's VM network, so VMs on different mini PCs can talk to each other directly.
- My home router doesn't know about these VM networks, and it doesn't let me add that information. So if a VM sends traffic to the router, another device at home, or the internet, the reply has nowhere to go.
- The fix for that is **NAT** (masquerading): the mini PC swaps the VM's address for its own Wi-Fi address on the way out, and swaps it back when the reply comes in.
- But NAT must **not** be used between the two VM networks. Kubernetes needs each node to see the other nodes' real addresses.

### What was already set up

Before changing anything, I checked what was already on the mini PCs:

- libvirt (the VM software) had already put the VM network into its own firewall group (zone), called `libvirt-routed`.
- libvirt had also added firewall rules that let traffic flow in and out of that group. So **VMs on different mini PCs could already reach each other.** No extra rule was needed for that.
- Nothing was doing NAT yet. That was the missing piece.
- The Wi-Fi connection is in a firewall zone called `FedoraServer` on both mini PCs. The Wi-Fi interface has a different name on each one (`wlp98s0` and `wlp2s0`).

## What I decided

Add one small firewall policy of my own, called `k8s-vm-nat`, and leave libvirt's rules alone.

In plain words, the policy says: **"For traffic going from the VMs out through Wi-Fi, apply NAT, unless it's going to the other VM network."**

| Setting | Value | What it means |
|---|---|---|
| Ingress zone | `libvirt-routed` | Only applies to traffic coming from the VMs |
| Egress zone | `FedoraServer` | Only applies to traffic leaving through Wi-Fi. Using the zone means I don't have to hard-code the different Wi-Fi interface names. |
| Rule | NAT if the source is this mini PC's VM network and the destination is **not** `10.10.0.0/16` | `10.10.0.0/16` covers both VM networks, so VM-to-VM traffic is left alone |
| Target | `CONTINUE` | My policy only adds NAT. It doesn't decide what's allowed or blocked; libvirt's rules still do that. |
| Priority | `-10` | Makes my policy run before libvirt's (`-1`), so the NAT rule is always checked |

### How it's applied

- Ansible copies the policy as a file into firewalld's saved settings (`/etc/firewalld/policies/k8s-vm-nat.xml`). Saved settings are loaded at every boot, so the rule survives a reboot.
- Before reloading the firewall, Ansible checks the settings are valid (`firewall-cmd --check-config`). If the file is broken, the firewall isn't reloaded.
- New setting for the whole lab: `vm_supernet: 10.10.0.0/16` in `group_vars/hypervisors.yml`.

## Other options I didn't choose

- **Change libvirt's own rules.** libvirt owns them and could overwrite my changes when it's updated.
- **Turn NAT on for all Wi-Fi traffic.** That would also NAT traffic between the VM networks, which would break Kubernetes.
- **Create the policy with a series of commands.** Keeping the whole policy in one file is easier to read, review in Git, and run again safely.
- **Tell the home router about the VM networks.** My ISP router doesn't allow this.
- **Put the VMs directly on the home network (bridging).** This doesn't work over Wi-Fi.

## What this means going forward

- ✅ VMs on different mini PCs see each other's real addresses. Tested and confirmed with `tcpdump`.
- ✅ VMs can reach the router, other home devices and the internet, appearing as the mini PC's address. Tested and confirmed with `tcpdump`.
- ⚠️ Traffic from a VM to the *other* mini PC's Wi-Fi address is also NAT'd. That's fine for now. Changing it would need a more complex rule.
- ⚠️ The policy depends on the zone names `libvirt-routed` and `FedoraServer`. If either one changes, the policy stops working and must be updated.
- ⚠️ This only covers IPv4.