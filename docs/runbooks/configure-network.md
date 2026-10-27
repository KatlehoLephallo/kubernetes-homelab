# Runbook: Configure the routed libvirt network (`k8s-net`)

## Purpose

Each hypervisor gets a routed libvirt network called `k8s-net`. Kubernetes VMs attach
to it and get addresses from that host's VM subnet. Route mode (rather than NAT or a
bridge onto the LAN) is used because the hosts are Wi-Fi only: Wi-Fi interfaces can't
be bridged, and routing keeps each VM's real 10.10.x.x address visible to the other host.

Addresses come from [`docs/network.md`](../network.md), which is the source of truth.

| Host | Subnet | Gateway (bridge IP) | DHCP range | Static range |
|------|--------|---------------------|------------|--------------|
| kvm1 | 10.10.1.0/24 | 10.10.1.1 | .100 – .199 | .2 – .99 |
| kvm2 | 10.10.2.0/24 | 10.10.2.1 | .100 – .199 | .2 – .99 |

## Files involved

| File | Role |
|------|------|
| `ansible/templates/k8s-net.xml.j2` | Network definition template |
| `ansible/inventory/host_vars/kvm1.yml`, `kvm2.yml` | Per-host values: `vm_subnet`, `vm_gateway`, `vm_uplink` |
| `ansible/playbooks/hypervisors/configure-libvirt-network.yml` | Applies the network to both hypervisors |
| `ansible/collections/requirements.yml` | Collections needed: `community.libvirt`, `ansible.posix` |

## Prerequisites

- Virtualisation installed on both hosts (`install-virtualisation.yml`)
- Ansible on the control machine can reach the `hypervisors` group:
  `ansible hypervisors -m ping`
- Collections installed:
  `ansible-galaxy collection install -r ansible/collections/requirements.yml`

## The network template explained

```xml
<network>
  <name>k8s-net</name>
  <forward mode='route' dev='{{ vm_uplink }}'/>
  <bridge name='virbr-k8s' stp='on' delay='0'/>
  <ip address='{{ vm_gateway }}' prefix='{{ vm_subnet.split("/")[1] }}'>
    <dhcp>
      <range start='{{ net_base }}.100' end='{{ net_base }}.199'/>
    </dhcp>
  </ip>
</network>
```

| Element | What it does |
|---------|--------------|
| `<name>` | Network name used by `virsh` and VM definitions. Same on both hosts so VM configs are portable. |
| `<forward mode='route' dev=…>` | The host routes VM traffic without NAT. `dev` limits forwarding to the Wi-Fi uplink, so libvirt's firewall rules only allow traffic out that interface. Interface names differ per host (`wlp98s0` on kvm1, `wlp2s0` on kvm2). |
| `<bridge name='virbr-k8s'>` | The virtual switch VMs plug into. `stp='on' delay='0'` enables loop protection without the forwarding delay. |
| `<ip address=… prefix=…>` | The bridge's own IP, which is the VMs' default gateway. The prefix length is taken from `vm_subnet`, so subnet size is defined only in host_vars. |
| `<dhcp><range>` | libvirt's dnsmasq hands out .100–.199. Cluster nodes use static IPs from .2–.99; DHCP is only for temporary or test VMs. |

`net_base` is set at the top of the template from `vm_gateway` (for example `10.10.1.1`
becomes `10.10.1`). This assumes a /24 subnet.

## Procedure

1. Check the template renders correctly for each host:

   ```bash
   ansible kvm1 -m debug -a "msg={{ lookup('template', 'ansible/templates/k8s-net.xml.j2') }}"
   ansible kvm2 -m debug -a "msg={{ lookup('template', 'ansible/templates/k8s-net.xml.j2') }}"
   ```

2. Syntax check and run the playbook from the repo root:

   ```bash
   ansible-playbook ansible/playbooks/hypervisors/configure-libvirt-network.yml --syntax-check
   ansible-playbook ansible/playbooks/hypervisors/configure-libvirt-network.yml
   ```

   The playbook:
   - installs `python3-libvirt` and `python3-lxml` (needed by `community.libvirt` on the hosts)
   - sets `net.ipv4.ip_forward = 1` in `/etc/sysctl.d/90-ip-forward.conf`
   - defines `k8s-net` from the template, starts it, and enables autostart

3. Run it a second time. Expect `changed=0` on both hosts, which confirms the playbook
   is idempotent.

## Verification

```bash
ansible hypervisors -a "virsh net-list --all"
ansible hypervisors -a "ip -br addr show virbr-k8s"
ansible hypervisors -a "sysctl net.ipv4.ip_forward"
```

Expected:

- `k8s-net` shows `active`, Autostart `yes`, Persistent `yes`
- `virbr-k8s` has `10.10.1.1/24` on kvm1 and `10.10.2.1/24` on kvm2
- `net.ipv4.ip_forward = 1`

Reboot test:

```bash
ansible hypervisors -m ansible.builtin.reboot
```

Run the three checks again once the hosts are back. The output should be the same.

## Changing the network?

Edit `docs/network.md` first, then the host_vars, then rerun the playbook.
libvirt only applies a changed definition when the network restarts:

```bash
ansible hypervisors -a "virsh net-destroy k8s-net"
ansible hypervisors -a "virsh net-start k8s-net"
```

VMs attached to `k8s-net` lose connectivity while it is down.

NB: My syytems for some reasoon don't keep the libvirt service on. So i have to start them every reboot using `ansible/playbooks/hypervisors/install-virtualisation.yml` playbook

## Rollback

Stop and remove the network on both hosts:

```bash
ansible hypervisors -a "virsh net-destroy k8s-net"
ansible hypervisors -a "virsh net-undefine k8s-net"
```

Forwarding stays enabled. Remove `/etc/sysctl.d/90-ip-forward.conf` too only if nothing
else on the host needs it (the libvirt `default` network also relies on forwarding).

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `virbr-k8s` shows `DOWN` | Normal while no VM is attached. The IP is what matters. |
| Ad-hoc `virsh` / `ip` commands report `CHANGED` | Normal for the `command` module, which can't tell whether a command changed anything. |
| Forwarding task reports `changed` on one host only | That host's forwarding setting came from a different file or formatting. Find duplicates with `grep -rs ip_forward /etc/sysctl.conf /etc/sysctl.d/` and keep only the Ansible-managed file. (Seen on kvm2 on the first run.) |
| `virt_net` fails with an import error | `python3-libvirt` or `python3-lxml` missing on the host. |
| Network fails to start | Usually a wrong `vm_uplink` interface name. Check with `ip -br link` on the host. |

## Known follow-ups

Not covered by this runbook (see `docs/network.md`):

- Static routes between hosts so each can reach the other's VM subnet
- NAT masquerade on each host for VM internet egress (my home router can't route
  replies to 10.10.x.x)