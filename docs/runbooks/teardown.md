## Teardown and rebuild of cluster VMs

### What this does

`teardown-vms.yml` removes the cluster VMs so they can be rebuilt from scratch.

Ansible doesn't keep a record of what it created (unlike Terraform's state file),
so removing things needs its own playbook.

| Removed | Left alone |
|---|---|
| Each VM (powered off and undefined in libvirt) | The base image in `k8s-base-images` |
| Its disk (`<vm>.qcow2`) | The `k8s-net` network |
| Its cloud-init files and `seed.iso` | Static routes and firewall rules |
| | Any VM not in the `vms` list (NB: find in `kvm1.yml` and `kvm2.yml`) |
| | The hosts themselves |

### When to use it

- A VM is broken and it's quicker to rebuild than to fix
- The cloud-init templates or VM settings changed and the VMs need recreating
- Proving the lab can be rebuilt from the repo

### Before you start

- **Everything on the VMs is lost.** Nothing on their disks is kept.
- Once the VMs are Kubernetes nodes, remove them from the cluster first.

### Tear down

The playbook won't run without the confirm flag. It deletes disks, so it shouldn't run by accident.

```bash
ansible-playbook ansible/playbooks/hypervisors/teardown-vms.yml -e confirm_teardown=yes
```

Without the flag, it stops on the first task 

**Healthy result**, from a test run with worker2 already shut off:

| Task | kvm1 | kvm2 |
|---|---|---|
| Power off running cluster VMs | worker1 | cp1 |
| Remove cluster VMs from libvirt | worker1, worker2 | cp1 |
| Delete VM folders | worker1, worker2 | cp1 |

VMs that are already off are skipped at power off, but still removed.
**Running it again** doesn't change anything.

### Verify the teardown

```bash
ansible hypervisors -a "virsh list --all"
ansible hypervisors -a "virsh net-list --all"
ansible hypervisors -a "ls -l /var/lib/libvirt/images"
ansible hypervisors -a "ip route"
```

- No cp1, worker1 or worker2. control, vm-2 and vm-test are still on kvm1.
- `k8s-net` is `active` on both hosts.
- `k8s-base-images` is still in `/var/lib/libvirt/images`.
- Static routes are still there: kvm1 → `10.10.2.0/24 via 192.168.18.202`,
  kvm2 → `10.10.1.0/24 via 192.168.18.201`.

### Rebuild

**1. Create the VMs:**

```bash
ansible-playbook ansible/playbooks/hypervisors/provision-vms.yml
```

Every task should show `changed` on both hosts.

**2. Wait about a minute for cloud-init, then check if they're up:**

```bash
ansible hypervisors -a "virsh list --all"
ansible kvm1 -a "ping -c 2 10.10.1.11"
ansible kvm1 -a "ping -c 2 10.10.1.12"
ansible kvm2 -a "ping -c 2 10.10.2.10"
```

**3. Forget the old SSH host keys on the control machine.** Rebuilt VMs have new keys,
so SSH refuses to connect ("host identification has changed") until the old ones are removed:

```bash
ssh-keygen -R 10.10.1.11
ssh-keygen -R 10.10.1.12
ssh-keygen -R 10.10.2.10
```

**4. Check SSH and cloud-init** (answer `yes` to the new-key prompt):

```bash
ssh -J ansible@kvm1 core@10.10.1.11 "hostname; cloud-init status"
ssh -J ansible@kvm1 core@10.10.1.12 "hostname; cloud-init status"
ssh -J ansible@kvm2 core@10.10.2.10 "hostname; cloud-init status"
```

Each should show its own hostname and `status: done`.

### Things to know

- **The `vms` list in host_vars decides what gets removed.** A VM taken out of that list
  won't be torn down. Remove the VM first, then take it out of the list.
- **Safety check on the delete path.** VM folders sit next to `k8s-base-images` in
  `/var/lib/libvirt/images`. If a VM name or `vm_image_dir` were empty, the delete would
  aim at the whole folder, so the playbook refuses to run if either is empty.
- **`destroy` pulls the plug.** It's not a clean shutdown, but the disks are deleted
  straight after, so there's nothing to protect, and it can't hang on a VM that won't stop.
- **`linkdown` on `virbr-k8s`** in `ip route` is normal when no VMs are running.
  Nothing is plugged into the virtual switch. It clears when the VMs start.
- **Autostart** comes back on with every rebuild, because `provision-vms.yml` sets it.
- **Storage pool list:** `virsh vol-list default` may still show old disks for a while.
  It's an out-of-date cached list, not real files. The next `virt-install` refreshes it.
