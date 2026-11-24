# Kubernetes and CRI-O packages on cluster nodes

This runbook covers installing, checking and upgrading the container runtime (CRI-O) and the Kubernetes tools (kubelet, kubeadm, kubectl) on `cp1`, `worker1` and `worker2`.

- **Playbook:** `ansible/playbooks/cluster-nodes/install-k8s-packages.yml`
- **Targets:** the `nodes` inventory group
- **Version variable:** `k8s_version` in `ansible/inventory/group_vars/nodes.yml` (currently `"1.36"`)

---

## What the playbook does

1. Adds `excludepkgs=kubernetes*,cri-o*,cri-tools*` to the `[main]` section of `/etc/dnf/dnf.conf`. This stops a normal `dnf upgrade` from touching these packages.
2. Installs these packages from Fedora's own repos, skipping the excludes for this one step only:

   | Package | Gives you |
   |---|---|
   | `kubernetes1.36` | kubelet (the node agent) |
   | `kubernetes1.36-kubeadm` | kubeadm (the cluster setup tool) |
   | `kubernetes1.36-client` | kubectl |
   | `cri-o1.36` | CRI-O (the container runtime) |
   | `cri-tools1.36` | crictl (CLI for talking to CRI-O) |
   | `container-selinux` | SELinux rules that let containers run with SELinux `Enforcing` |

3. Enables and starts `crio`.
4. Enables `kubelet` (see [Expected behaviour](#expected-behaviour-before-kubeadm-init)).

**Left out on purpose:** `kubernetes1.36-systemd`. It runs the control-plane parts (API server, scheduler, and so on) as normal host services. kubeadm doesn't use it. It runs those parts as containers (static pods) instead.

---

## Before you start

- The kernel prerequisites playbook (`kernel-prereqs.yml`) has already been run on the nodes.
- The nodes answer Ansible from the control machine: `ansible nodes -m ping`
- Run all commands from the repo root on the control machine.

---

## Running the playbook

```bash
ansible-playbook ansible/playbooks/cluster-nodes/install-k8s-packages.yml
```

- First run: expect `changed` on the dnf.conf, install and service tasks.
- Any run after that: expect `changed=0` on all three nodes. If something shows `changed` on a later run, something on the node drifted (for example, someone edited `dnf.conf` or stopped `crio`), and the playbook has just put it back.

---

## Checking the result

```bash
# Packages at the expected version
ansible nodes -a "rpm -q kubernetes1.36 kubernetes1.36-kubeadm kubernetes1.36-client cri-o1.36 cri-tools1.36"

# Excludes are in place
ansible nodes -a "grep excludepkgs /etc/dnf/dnf.conf"

# A normal upgrade would not touch our packages
ansible nodes -m ansible.builtin.shell -a "dnf upgrade --assumeno 2>&1 | grep -Ei 'kubernetes|cri-o|cri-tools' || echo 'PASS: none listed'"

# CRI-O running and ready
ansible nodes -m ansible.builtin.shell -a "systemctl is-active crio && crictl info

# kubelet enabled
ansible nodes -a "systemctl is-enabled kubelet"

# SELinux still on
ansible nodes -a "getenforce"
```

**Good result:** the 1.36.x versions on every node, the `excludepkgs` line, `PASS`, `active` plus `"status": true`, `enabled`, `Enforcing`.

---

## Working with the excludes in place

The excludes hide these packages from dnf. That's the point, but it changes how normal dnf commands behave on the nodes:

| You run | What happens |
|---|---|
| `dnf upgrade` | Updates everything **except** Kubernetes, CRI-O and cri-tools. |
| `dnf list kubernetes1.36*` | `No matches found.` This is expected. It doesn't mean the repo is broken. |
| `dnf install kubernetes1.36-...` | Fails with no match, for the same reason. |

To get past the excludes **for one command only**, add:

```bash
--setopt=disable_excludes=main
```

### Checking whether newer patch versions are available

```bash
ansible nodes -a "dnf list --upgrades --setopt=disable_excludes=main kubernetes1.36* cri-o1.36* cri-tools1.36*"
```

---

## Expected behaviour before kubeadm init

**kubelet is enabled but inactive right after install.** Enabling a service only means "start at boot". Nothing has started kubelet yet.

```text
Active: inactive (dead)
```

**After a reboot, kubelet restarts in a loop.** kubelet starts at boot, can't find its config file (`/var/lib/kubelet/config.yaml`), exits, and systemd tries again every few seconds. That file is only written by `kubeadm init` (on cp1) or `kubeadm join` (on the workers), and kubeadm starts kubelet itself at that point.

**Both of these are normal until the node has been initialised or joined.** Don't try to "fix" kubelet before then.

---

## Why the install step uses the `dnf` command, not the `dnf` module

The nodes run Ansible with `python3.14` (`ansible_python_interpreter`) because Fedora 45's default Python (3.15) breaks this Ansible version. The `ansible.builtin.dnf` module needs dnf5's Python library (`python3-libdnf5`), and Fedora only builds that for the system Python. So the module fails with:

```text
Could not import the libdnf5 python module using /usr/bin/python3.14
```

The playbook runs `dnf install -y --setopt=disable_excludes=main ...` through `ansible.builtin.command`. It reports `changed` only when dnf's output doesn't contain `Nothing to do`.

**This is temporary.** When Ansible supports Python 3.15 and the Python 3.14 workaround is removed (follow-up ticket), switch this task back to `ansible.builtin.dnf` with `state: present` and `disable_excludes: main`.

---

## Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| `Could not import the libdnf5 python module` | Someone switched the install task to the `dnf` module while the Python 3.14 workaround is still in place | Use the `dnf` command task (see above) |
| `No matches found` when installing or listing by hand | The excludes are working | Add `--setopt=disable_excludes=main` |
| Install task shows `changed` on every run | dnf's "Nothing to do" wording changed | Run the same `dnf install` by hand on a node that already has the packages, and update `changed_when` to match the new text |
| `crio` not active | CRI-O failed to start | `ansible <node> -a "journalctl -u crio -n 50 --no-pager"` |
| `crictl` warns about "default endpoints" | No `/etc/crictl.yaml`, so crictl guesses the runtime socket | Harmless. It still finds CRI-O |
| `getenforce` shows `Permissive` or `Disabled` | Someone changed SELinux on the node | Not expected. Investigate before going further. The cluster is designed to run with SELinux `Enforcing` |
| A Kubernetes/CRI-O package was upgraded by accident | The excludes were removed or bypassed | Check `/etc/dnf/dnf.conf`, re-run the playbook to restore the excludes, and check versions with `rpm -q` |