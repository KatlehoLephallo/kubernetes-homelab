# Runbook: Host bootstrap

One-time manual setup for a new KVM host, so Ansible can manage it.
Run these steps from the control machine for each host.

## Prerequisites

- The host is installed and reachable on the LAN (see the network plan for its IP)
- You can SSH to the host with your own account, and that account has sudo
- An SSH key exists on the control machine (`~/.ssh/<key>`)

## 1. Create the `ansible` user

```bash
ssh <host> 'id ansible || sudo useradd -m -s /bin/bash ansible'
ssh -t <host> 'sudo passwd ansible'
```

The password is only needed for `ssh-copy-id` in the next step.

## 2. Copy the SSH key

Password logins must be enabled on the host for this step:

```bash
ssh <host> 'sudo sshd -T | grep -i passwordauthentication'   # expect "yes"
ssh-copy-id -i ~/.ssh/key.pub ansible@<host-ip>
```

Check that login works without a password:

```bash
ssh ansible@<host-ip> whoami   # expect "ansible"
```

## 3. Grant passwordless sudo

```bash
ssh -t <host> "echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible && sudo visudo -cf /etc/sudoers.d/ansible"
```

Expect `parsed OK`. The filename must not contain a dot, because sudo ignores
files in `sudoers.d/` that do.

## 4. Add the host to SSH config

Add a block to `~/.ssh/config` on the control machine:

```
Host <host>
    HostName <host-ip>
    User ansible
    IdentityFile ~/.ssh/<key>
    IdentitiesOnly yes
```

## 5. Add the host to the inventory

- Add the host name under `[hypervisors]` in `ansible/inventory/hosts.ini`
- Create `ansible/inventory/host_vars/<host>.yml` with its `host_ip`, `vm_subnet`
  and `vm_gateway`, matching the network plan

## 6. Verify

From the repo root:

```bash
ssh <host> 'sudo -n true && echo sudo-ok'
ansible-playbook ansible/playbooks/verify.yml
```

Expect `sudo-ok`, then `failed=0` for the host in the PLAY RECAP.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `UNREACHABLE` | SSH config alias missing or wrong. Test `ssh <host>` directly. |
| `Missing sudo password` | Sudoers drop-in missing, misnamed, or invalid. Re-run step 3. |
| Key refused after `ssh-copy-id` | Wrong permissions on `~ansible/.ssh`. On RHEL/Fedora, also run `sudo restorecon -R /home/ansible/.ssh`. |