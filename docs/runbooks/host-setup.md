# Hypervisor host setup

## Purpose

Prepare the hypervisors (kvm1 and kvm2) to run VMs: install KVM/QEMU and libvirt,
enable libvirt's modular daemons, and confirm `virsh` works.

## Scope

| Item | Value |
|---|---|
| Hosts | `kvm1` (192.168.18.201), `kvm2` (192.168.18.202) |
| Inventory group | `hypervisors` |
| OS | Fedora 44 on both hosts |
| Playbook | `ansible/playbooks/hypervisors/install-virtualisation.yml` |
| Run from | Repo root on the control machine (so `ansible.cfg` is picked up) |

## Prerequisites

- Both hosts run the same Fedora release. Mismatched releases mean mismatched
  libvirt and QEMU versions, which may causes problems later.
- Ansible can reach both hosts as the `ansible` user, with passwordless sudo.
- `become=True` is set in `ansible.cfg`, so the playbook runs tasks as root.

## Pre-flight checks

Run these from the repo root before the first run on a new host.

```bash
ansible hypervisors -m ping
ansible hypervisors -m shell -a "ls -l /dev/kvm"
ansible hypervisors -m setup -a "filter=ansible_distribution*"
```

| Check | Healthy result | If not |
|---|---|---|
| `ping` | `"ping": "pong"` from every host | Fix SSH access or the inventory before going further |
| `/dev/kvm` | The file exists, owned by the `kvm` group | Missing means the KVM kernel module isn't loaded |
| Distribution | `Fedora`, same major version on both hosts | Upgrade the older host |

Target the `hypervisors` group, not `all`. The `all` group includes cluster VMs
that may not exist yet, and they would show as unreachable.


## Run

```bash
ansible-playbook ansible/playbooks/hypervisors/install-virtualisation.yml --syntax-check
ansible-playbook ansible/playbooks/hypervisors/install-virtualisation.yml
```

Expected on a fresh host: the install task reports `changed`, the socket task
reports `ok` (Fedora enables the sockets on install), the verify task reports `ok`,
and the recap shows `failed=0`.

## What it installs and why

### Packages

| Package | Why |
|---|---|
| `qemu-kvm` | The hypervisor that runs the VMs |
| `libvirt` | The management layer: daemons, drivers and config for VMs, networks and storage |
| `libvirt-client` | Provides `virsh`, the CLI used to manage and verify libvirt |
| `virt-install` | CLI for creating VMs |
| `python3-libvirt` | Python bindings that Ansible's `community.libvirt` modules need on the host |


### Sockets

libvirt runs as modular daemons, one per job (see ADR 0001). Each daemon is started
on demand by a systemd socket, so the playbook enables the sockets rather than the
daemons. A daemon only runs while something is talking to it.

| Socket | Daemon it starts | Manages |
|---|---|---|
| `virtqemud.socket` | `virtqemud` | VMs (what `virsh list` talks to) |
| `virtnetworkd.socket` | `virtnetworkd` | Virtual networks |
| `virtstoraged.socket` | `virtstoraged` | Storage pools and disk images |
| `virtnodedevd.socket` | `virtnodedevd` | Host devices (e.g. passing a device through to a VM) |
| `virtsecretd.socket` | `virtsecretd` | Secrets such as disk encryption keys |
| `virtnwfilterd.socket` | `virtnwfilterd` | Firewall rules applied to VM network traffic |

Fedora enables these sockets automatically when libvirt is installed. The playbook
enables them anyway, so the desired state is guaranteed rather than left to package
defaults.

## Verify

The playbook's last task runs `virsh list --all` and fails if libvirt doesn't
respond. To check by hand:

```bash
ansible hypervisors -m command -a "virsh --connect qemu:///system list --all"
```


## Re-running

The playbook is idempotent and safe to re-run at any time. On hosts that are already
configured, the recap should show `changed=0`. Anything else means the host has
drifted from the desired state, or a task isn't written idempotently. Investigate
before moving on.

## Troubleshooting and notes

- **Package module.** The plain `ansible.builtin.dnf` module works on Fedora 44. If a
  future Fedora release fails with an error about missing DNF Python bindings, switch
  to `ansible.builtin.dnf5`.
- **Existing libvirt network on kvm1.** The `default` network (`192.168.122.0/24`) is
  in use by the pre-existing VMs. Project VM subnets must not overlap with it.

## Related

- `network.md`: IP plan (source of truth for host and VM subnets)