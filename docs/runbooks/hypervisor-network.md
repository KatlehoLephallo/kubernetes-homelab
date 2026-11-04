# Runbook: Hypervisor networking

How VM traffic gets onto and between the hypervisors (kvm1 and kvm2).

| Section | What it covers |
|---------|----------------|
| [1. Routed libvirt network (`k8s-net`)](#1-routed-libvirt-network-k8s-net) | The VM subnet and gateway on each host |
| [2. Static routes between hosts](#2-static-routes-between-hosts) | How each host reaches the other host's VM subnet |

Addresses come from [`docs/network.md`](../network.md), which is the source of truth.

| Host | Wi-Fi IP | VM subnet | Gateway (bridge IP) | DHCP range | Static range |
|------|----------|-----------|---------------------|------------|--------------|
| kvm1 | 192.168.18.201 | 10.10.1.0/24 | 10.10.1.1 | .100 – .199 | .2 – .99 |
| kvm2 | 192.168.18.202 | 10.10.2.0/24 | 10.10.2.1 | .100 – .199 | .2 – .99 |

## Prerequisites (both sections)

- Virtualisation installed and libvirt's modular daemons enabled on both hosts
  (`install-virtualisation.yml`, see `host-setup.md`)
- Ansible on the control machine can reach the `hypervisors` group:
  `ansible hypervisors -m ping`
- Collections installed:
  `ansible-galaxy collection install -r ansible/collections/requirements.yml`
- All commands run from the repo root, so `ansible.cfg` is picked up

---

## 1. Routed libvirt network (`k8s-net`)

### Purpose

Each hypervisor gets a routed libvirt network called `k8s-net`. Kubernetes VMs attach
to it and get addresses from that host's VM subnet. Route mode (rather than NAT or a
bridge onto the LAN) is used because the hosts are Wi-Fi only: Wi-Fi interfaces can't
be bridged, and routing keeps each VM's real 10.10.x.x address visible to the other host.

### Files involved

| File | Role |
|------|------|
| `ansible/templates/k8s-net.xml.j2` | Network definition template |
| `ansible/inventory/host_vars/kvm1.yml`, `kvm2.yml` | Per-host values: `vm_subnet`, `vm_gateway`, `vm_uplink` |
| `ansible/playbooks/hypervisors/configure-libvirt-network.yml` | Applies the network to both hypervisors |

### The network template explained

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
| `<bridge name='virbr-k8s'>` | The virtual switch VMs plug into (15-character limit on bridge names). `stp='on' delay='0'` enables loop protection without the usual forwarding delay. |
| `<ip address=… prefix=…>` | The bridge's own IP, which is the VMs' default gateway. The prefix length is taken from `vm_subnet`, so subnet size is defined only in host_vars. |
| `<dhcp><range>` | libvirt's dnsmasq hands out .100–.199. Cluster nodes use static IPs from .2–.99; DHCP is only for temporary or test VMs. |

`net_base` is set at the top of the template from `vm_gateway` (for example `10.10.1.1`
becomes `10.10.1`). This assumes a /24 subnet.

### Procedure

1. Check the template renders correctly for each host (changes nothing on the hosts):

   ```bash
   ansible kvm1 -m debug -a "msg={{ lookup('template', 'ansible/templates/k8s-net.xml.j2') }}"
   ansible kvm2 -m debug -a "msg={{ lookup('template', 'ansible/templates/k8s-net.xml.j2') }}"
   ```

2. Syntax check and run the playbook:

   ```bash
   ansible-playbook ansible/playbooks/hypervisors/configure-libvirt-network.yml --syntax-check
   ansible-playbook ansible/playbooks/hypervisors/configure-libvirt-network.yml
   ```

   The playbook:
   - installs `python3-libvirt` and `python3-lxml` (needed by `community.libvirt` on the hosts)
   - sets `net.ipv4.ip_forward = 1` in `/etc/sysctl.d/90-ip-forward.conf`
   - defines `k8s-net` from the template, starts it, and enables autostart

3. Run it a second time. Expect `changed=0` on both hosts.

### Verification

Check the bridge **before** running any `virsh` command. `virsh` wakes the libvirt
daemons, which autostart networks as a side effect and can hide a boot-time problem.

```bash
ansible hypervisors -a "ip -br addr show virbr-k8s"
ansible hypervisors -a "sysctl net.ipv4.ip_forward"
ansible hypervisors -a "virsh net-list --all"
```

Expected:

- `virbr-k8s` has `10.10.1.1/24` on kvm1 and `10.10.2.1/24` on kvm2
- `net.ipv4.ip_forward = 1`
- `k8s-net` shows `active`, Autostart `yes`, Persistent `yes`

Reboot test:

```bash
ansible hypervisors -m ansible.builtin.reboot
ansible hypervisors -a "ip -br addr show virbr-k8s"
```

The bridge IPs should be there straight after boot, with no `virsh` command run first.

### Changing the network

Edit `docs/network.md` first, then the host_vars or template, then rerun the playbook.
libvirt only applies a changed definition when the network restarts:

```bash
ansible hypervisors -a "virsh net-destroy k8s-net"
ansible hypervisors -a "virsh net-start k8s-net"
```

VMs attached to `k8s-net` lose connectivity while it is down.

### Rollback

```bash
ansible hypervisors -a "virsh net-destroy k8s-net"
ansible hypervisors -a "virsh net-undefine k8s-net"
```

Forwarding stays enabled. Remove `/etc/sysctl.d/90-ip-forward.conf` too only if nothing
else on the host needs it (the libvirt `default` network also relies on forwarding).

### Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `virbr-k8s` shows `DOWN` / `linkdown` | Normal while no VM is attached. The IP is what matters. |
| Bridge missing after boot, appears once `virsh` is run | `virtnetworkd.service` isn't enabled, so nothing starts libvirt at boot. Enabling only the sockets isn't enough for autostart. Fixed in `install-virtualisation.yml` (see `host-setup.md`). |
| Ad-hoc `virsh` / `ip` commands report `CHANGED` | Normal for the `command` module, which can't tell whether a command changed anything. |
| Forwarding task reports `changed` on one host only | That host's forwarding setting came from a different file or formatting. Find duplicates with `grep -rs ip_forward /etc/sysctl.conf /etc/sysctl.d/` and keep only the Ansible-managed file. (Seen on kvm2 on the first run.) |
| Play does nothing, warning "no hosts matched" | The play targets `hypervisors`; check the group name in `inventory.ini`. Confirm with `--list-hosts`. |
| `virt_net` fails with an import error | `python3-libvirt` or `python3-lxml` missing on the host. |
| Network fails to start | Usually a wrong `vm_uplink` interface name. Check with `ip -br link` on the host. |
| Host doesn't come back after reboot | Wi-Fi credentials may be stored per user instead of system-wide in NetworkManager, so the host can't connect until someone logs in. |

---

## 2. Static routes between hosts

### Purpose

Each host knows its own VM subnet and the home LAN. Anything else goes to the home
router, which doesn't know where 10.10.x.x is and doesn't support static routes. So
each host needs a route telling it to send the other host's VM subnet to that host's
Wi-Fi IP. The other host then forwards the traffic onto its own `k8s-net` bridge.

| Host | Route |
|------|-------|
| kvm1 | `10.10.2.0/24 via 192.168.18.202` |
| kvm2 | `10.10.1.0/24 via 192.168.18.201` |

Both routes are needed: one carries traffic there, the other carries replies back.
This works because both hosts are on the same LAN and reach each other directly; the
router only passes Wi-Fi frames between them and never routes 10.10.x.x traffic.

### Why NetworkManager, not `ip route add`

`ip route add` only changes the live routing table and is lost on reboot. The routes
are saved on the `home-wifi` NetworkManager profile instead, so NetworkManager adds
them every time the profile connects, including at boot.

### Files involved

| File | Role |
|------|------|
| `ansible/playbooks/hypervisors/configure-host-routes.yml` | Adds the routes to the profile and applies them |
| `ansible/inventory/host_vars/kvm1.yml`, `kvm2.yml` | `wifi_profile` (profile name), `vm_subnet`, `host_ip`, `vm_uplink` |

### Before you start: one Wi-Fi profile per host

Routes live on a profile, not an interface. If a host has two profiles for the same
Wi-Fi network, it can connect with the one that has no routes, and the routes go
missing after some reboots. Check each host has exactly one, and that it is
system-wide (empty `permissions`, so it connects at boot without a login):

```bash
ansible hypervisors -a "nmcli -f NAME,UUID,DEVICE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show"
ansible hypervisors -a "nmcli -g connection.permissions connection show home-wifi"
```

One-time cleanup done in Oct 2026 (kvm1 had a stale duplicate, `SW-1006-9271`,
alongside the active `SW-1006-9271 1`):

```bash
# Delete the inactive duplicate by UUID (UUID avoids confusing similar names)
ansible kvm1 -a "nmcli connection delete <stale-profile-uuid>"

# Rename the active profile on both hosts. Renaming doesn't reconnect the Wi-Fi.
ansible kvm1 -a "nmcli connection modify <active-profile-uuid> connection.id home-wifi"
ansible kvm2 -a "nmcli connection modify SW-1006-9271 connection.id home-wifi"
```

### How the playbook works

- **Routes come from the inventory.** For each host, the playbook loops over every
  other host in `hypervisors` and builds `<their vm_subnet> <their host_ip>`. No route
  variables are stored anywhere. A new hypervisor gets routes to and from every other
  host automatically.
- **It runs `nmcli` through the `command` module**, not `community.general.nmcli`.
  The module manages the whole Wi-Fi connection (SSID and related settings), and a
  mistake there could cut off the connection Ansible is using. `nmcli connection
  modify ... +ipv4.routes` only touches routes, and `+` adds rather than replaces.
- **Idempotency is handled by hand.** The playbook reads the routes already saved on
  the profile and only adds a route if it isn't there, so a second run reports
  `changed=0`.
- **A handler applies the change with `nmcli device reapply`.** Editing a profile
  doesn't change the live connection. `reapply` pushes the new settings to the running
  connection without reconnecting the Wi-Fi, so SSH stays up. (`nmcli connection up`
  would reconnect and briefly drop SSH.) The handler only runs when a route was added.

### Procedure

```bash
ansible-playbook ansible/playbooks/hypervisors/configure-host-routes.yml --syntax-check
ansible-playbook ansible/playbooks/hypervisors/configure-host-routes.yml
ansible-playbook ansible/playbooks/hypervisors/configure-host-routes.yml
```

First run: the add-route task and the handler report `changed` on both hosts.
Second run: the add-route task is `skipping` and the recap shows `changed=0`.

### Verification

All checks are read-only.

**1. Routes are saved on the profile:**

```bash
ansible hypervisors -a "nmcli -g ipv4.routes connection show home-wifi"
```

Expect `10.10.2.0/24 192.168.18.202` on kvm1 and `10.10.1.0/24 192.168.18.201` on kvm2.

**2. Routes are in the live routing table:**

```bash
ansible hypervisors -a "ip route show root 10.10.0.0/16"
```

Expected on kvm1 (kvm2 is the mirror):

```
10.10.1.0/24 dev virbr-k8s proto kernel scope link src 10.10.1.1 linkdown
10.10.2.0/24 via 192.168.18.202 dev wlp98s0 proto static metric 600
```

`proto static` means NetworkManager added the route from the profile.

**3. The kernel chooses the route:**

```bash
ansible kvm1 -a "ip route get 10.10.2.1"
ansible kvm2 -a "ip route get 10.10.1.1"
```

Expect `via 192.168.18.202 dev wlp98s0` on kvm1 and `via 192.168.18.201 dev wlp2s0`
on kvm2. `via 192.168.18.1` means traffic is going to the router and the route is missing.

**4. Traffic gets there and back:**

```bash
ansible kvm1 -a "ping -c 3 10.10.2.1"
ansible kvm1 -a "ping -c 3 -I 10.10.1.1 10.10.2.1"
ansible kvm2 -a "ping -c 3 10.10.1.1"
ansible kvm2 -a "ping -c 3 -I 10.10.2.1 10.10.1.1"
```

- A plain ping is sent from the host's Wi-Fi IP, so the reply comes straight back over
  the LAN. It only tests the sending host's route.
- A ping with `-I <own bridge IP>` makes the other host reply to a 10.10.x.x address,
  which it can only reach through its own route. It tests both routes at once.

Latency varies a lot (roughly 3–95 ms) because the hosts are on Wi-Fi. Packet loss
is what matters, and it should be 0%.

**5. Routes survive a reboot:**

```bash
ansible hypervisors -m ansible.builtin.reboot
ansible hypervisors -a "ip route show root 10.10.0.0/16"
```

Same output as before the reboot, without running Ansible again.

### Removing or changing a route

The playbook only adds routes. To remove one (for example after a subnet change),
remove it from the profile and reapply:

```bash
ansible kvm1 -a "nmcli connection modify home-wifi -ipv4.routes '10.10.2.0/24 192.168.18.202'"
ansible kvm1 -a "nmcli device reapply wlp98s0"
```

Then update `docs/network.md` and host_vars, and rerun the playbook to add the new route.

### Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `ip route show 10.10.0.0/16` prints nothing | Without `root`, `ip route show` only matches a route for exactly that prefix. Use `ip route show root 10.10.0.0/16`. |
| Route is on the profile but not in `ip route` | The change wasn't applied to the live connection. Run `nmcli device reapply <wifi interface>`. |
| Routes missing after some reboots | The host has a second profile for the same Wi-Fi without the routes. See "Before you start". |
| Second playbook run reports `changed` again | `nmcli` is printing the saved route in a different format from the string the playbook checks for. Compare with `nmcli -g ipv4.routes connection show home-wifi`. |
| Plain ping works, `-I` ping fails | The sending host's route is fine; either the other host's return route is missing or a firewall is blocking the forwarded reply. Check `ip route` on the other host, then firewall rules. |
| Ping fails entirely | Check `ip route get` on the sending host, and that the other host's bridge exists (`ip -br addr show virbr-k8s`). |

### Not yet tested

These checks reach the other host's bridge IP, which the other host answers itself.
Traffic passing *through* a host to a VM on its bridge can only be tested once VMs
exist. Do this as part of the VM-creation ticket.

---

## Known follow-ups

- **VM internet egress:** the router can't route replies to 10.10.x.x, so each host must
  masquerade VM traffic leaving for the internet. Exclude 10.10.0.0/16, 10.244.0.0/16
  and 10.96.0.0/12 from NAT (see `docs/network.md`). This will be section 3.