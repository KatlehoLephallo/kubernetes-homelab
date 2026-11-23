## Provisioning the cluster VMs

### What this does

Creates the three cluster VMs from a Fedora 45 cloud image, using Ansible and libvirt. cloud-init sets each VM's hostname, static IP, gateway, DNS, admin user and SSH key on first boot.

| VM | Host | IP | MAC |
|---|---|---|---|
| cp1 | kvm2 | 10.10.2.10 | 52:54:00:00:02:10 |
| worker1 | kvm1 | 10.10.1.11 | 52:54:00:00:01:11 |
| worker2 | kvm1 | 10.10.1.12 | 52:54:00:00:01:12 |

The IP plan's source of truth is [network.md](../network.md). Each VM's name, IP, MAC and sizes are set in the `vms` list in `inventory/host_vars/kvm1.yml` and `inventory/host_vars/kvm2.yml`.


### How to run

Run from the repo root on the control machine, in this order:

```bash
# 1. Install the tools and download the Fedora base image (once per host)
ansible-playbook ansible/playbooks/hypervisors/vm-base-image.yml

# 2. Create the VMs
ansible-playbook ansible/playbooks/hypervisors/provision-vms.yml
```

Both playbooks are safe to rerun. `provision-vms.yml` skips any VM that already exists, so a second run shows `changed=0`:


### What gets created

Each host has one shared base image, plus one folder per VM:

```
/var/lib/libvirt/images/
├── k8s-base-images/
│   └── Fedora-Base-image.qcow2   
└── worker1/
    ├── worker1.qcow2        
    ├── seed.iso             
    └── cloud-init/          
        ├── meta-data
        ├── user-data
        └── network-config
```


### How to reach the VMs

The home router doesn't know about the VM subnets, so the control machine can't reach the VMs directly. SSH goes through the VM's hypervisor as a jump host (`-J`):

```bash
ssh -J ansible@kvm1 core@10.10.1.11   # worker1 via kvm1
ssh -J ansible@kvm1 core@10.10.1.12   # worker2 via kvm1
ssh -J ansible@kvm2 core@10.10.2.10   # cp1 via kvm2
```

This needs `AllowTcpForwarding` to be on (the default) on both hypervisors.

### How to verify

**VMs are running:**

```bash
ansible hypervisors -a "virsh list --all"
```

worker1 and worker2 should be `running` on kvm1, and cp1 `running` on kvm2.

**Sizes match the variables:**

```bash
ansible kvm1 -a "virsh dominfo worker1"
```

```
CPU(s):         4
Max memory:     10485760 KiB
Used memory:    8388608 KiB
```

**Disk points to the base image:**

```bash
ansible kvm1 -a "qemu-img info /var/lib/libvirt/images/worker1/worker1.qcow2"
```

```
virtual size: 50 GiB (53687091200 bytes)
disk size: 196 KiB
backing file: /var/lib/libvirt/images/k8s-base-images/Fedora-Cloud-Base-Generic-45-1.4.x86_64.qcow2
```

**Hostname, cloud-init and internet access:**

```bash
ssh -J ansible@kvm1 core@10.10.1.11 "hostname; cloud-init status; ping -c 1 fedoraproject.org"
```

```
worker1
status: done
1 packets transmitted, 1 received, 0% packet loss
```



### How to rebuild a VM

A rebuild deletes the VM and everything on it, then creates a fresh one with the same name, IP and sizes. Example for worker2:

```bash
# 1. Power off
ansible kvm1 -a "virsh destroy worker2"

# 2. Remove it from libvirt
ansible kvm1 -a "virsh undefine worker2 --nvram"

# 3. Delete its folder
ansible kvm1 -m file -a "path=/var/lib/libvirt/images/worker2 state=absent"

# 4. Recreate it. Only worker2 should show changed.
ansible-playbook ansible/playbooks/hypervisors/provision-vms.yml

# 5. Forget the old VM's SSH identity on the control machine
ssh-keygen -R 10.10.1.12

# 6. Verify
ssh -J ansible@kvm1 core@10.10.1.12 "hostname; cloud-init status"
```

Both steps 2 and 3 are needed. If the VM still exists, the playbook skips it. If the folder still exists, the old disk is reused, and cloud-init won't run again on it.

NB: Once a node is part of a kubernetes cluster. It has to be removed the cluster before deleting it

### Things to know

- **Size variables don't resize existing VMs.** `vcpus`, `memory_mb` and `disk_gb` only apply when a VM is created. 
- **Max values allow growing a VM while it runs.** `vcpus_max` and `memory_max_mb` set how far vCPUs and RAM can be raised later without recreating the VM.
- **cloud-init only runs once.** Hostname, IP, user and SSH key are applied on first boot. Changing the templates later does nothing to existing VMs. Rebuild the VM instead.
- **Changing the Fedora version?:** update `fedora_image_url` and `fedora_image_checksum`, then rerun `vm-base-image.yml`. Save the new image next to the old one. Don't delete the old image while VMs use it as their backing file.

### Problems during setup

***Limited space where images are stored** I have other VMs running on kvm1 and they where taking up most of the space in /var/lib/libvirt/images

I had to expand the lvm to accomodate 2 new VMs. Space on VMs is limited on purpose. I want to manually attach disks when disks get full 

**SSH jump through kvm1 refused.** `ssh -J` through kvm1 failed with `channel 0: open failed: administratively prohibited`. kvm1's `/etc/ssh/sshd_config` had an active `AllowTcpForwarding no` line, kvm2 didn't. Removed the line on kvm1 and reloaded sshd.

This was changed by hand and isn't managed by a playbook.