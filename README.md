# kubernetes-homelab
Building a kubernetes cluster using kubeadm. One control plane and two workers across two KVM hosts, built with kubeadm and provisioned with Ansible

## Running the Ansible playbooks

### Requirements

- A macOS or Linux control machine
- Ansible and ansible-lint:

  ```bash
  brew install ansible ansible-lint
  ```

  Tested with ansible-core 2.21.5 (Ansible 14.5.0) and ansible-lint 26.10.1.
- Two KVM hosts, `kvm1` and `kvm2`, reachable on the LAN. Addresses are in the
  [network plan](<docs/network.md>).

### One-time host bootstrap (manual)

Before Ansible can manage the hosts, each one needs:

1. An `ansible` user with your SSH public key in `authorized_keys`
2. Passwordless sudo for that user (`/etc/sudoers.d/ansible`)

Full steps: [host bootstrap runbook](<docs/runbooks/host-bootstrap.md>).

### SSH config

Ansible connects using the host aliases `kvm1` and `kvm2`. Add these to
`~/.ssh/config` on the control machine:

```
Host kvm1
    HostName 192.168.18.201
    User ansible
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host kvm2
    HostName 192.168.18.202
    User ansible
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

### Quick start

Run everything from the repository root, so Ansible picks up `ansible.cfg`:

```bash
git clone https://github.com/KatlehoLephallo/kubernetes-homelab.git
cd kubernetes-homelab

ansible --version          # "config file" should point at this repo's ansible.cfg
ansible-inventory --graph  # shows the hypervisors and k8s_nodes groups
ansible hypervisors -m ping
ansible-playbook ansible/playbooks/verify.yml
```

`verify.yml` is the smoke test. It checks SSH connectivity and confirms that
privilege escalation reaches root. Run it after any host change.

### Repository layout

```
ansible.cfg                 # project config (inventory path, become settings)
.ansible-lint               # lint config, production profile
ansible/
  inventory/
    hosts.ini               # host and group names only
    group_vars/             # shared settings and cluster-wide IP ranges
    host_vars/              # per-host addresses and subnets
  playbooks/                # entry-point playbooks
  roles/                    # reusable roles
```

All addresses come from `group_vars` and `host_vars`, which mirror the
[network plan](<docs/network.md>). Nothing is hardcoded in playbooks or roles.

### Linting

```bash
ansible-lint
```

The repo uses the `production` profile. Note that ansible-lint only checks files
tracked by git.