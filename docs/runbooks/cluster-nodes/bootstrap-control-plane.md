## Bootstrapping the control plane

### What this does

Turns cp1 into the Kubernetes control plane with kubeadm, using the `bootstrap-control-plane.yml` playbook. The control plane is built entirely from the repo, and the playbook is safe to rerun.

It does not install the pod network (Calico) or join the workers. Those have their own runbooks.

| Setting | Value | Where it's set |
|---|---|---|
| API endpoint | `control-plane:6443` | `k8s_api_endpoint_name` in `group_vars/nodes.yml`; port 6443 in the template |
| Pod CIDR | `10.244.0.0/16` | `k8s_pod_cidr` in `group_vars/nodes.yml` |
| Service CIDR | `10.96.0.0/12` | `k8s_service_cidr` in `group_vars/nodes.yml` |
| Kubernetes version | the installed kubeadm's version | read from `kubeadm version` at run time |
| Container runtime | CRI-O (`unix:///var/run/crio/crio.sock`) | the template |

Files:

- Template: `ansible/playbooks/cluster-nodes/templates/kubeadm-config.yml.j2`
- Rendered config on cp1: `/root/kubeadm-config.yml`
- Admin kubeconfig on cp1: `/etc/kubernetes/admin.conf`, with a copy for `core` at `/home/core/.kube/config`

### Before you start

- `install-k8s-packages.yml` has run on all nodes. Besides the packages, it copies the containers-common defaults to `/etc/containers/` and removes CRI-O's default CNI configs. The CNI folder should be empty:

  ```bash
  ansible nodes -a "ls /etc/cni/net.d/"
  ```

- `vm-baseline.yml` has run, so every node can resolve the API endpoint name:

  ```bash
  ansible nodes -a "getent hosts control-plane"
  # 10.10.2.10      control-plane
  ```

- cp1 is either clean (never initialised, or reset) or a healthy control plane. See "How the playbook decides what to do" below.

### How to run

Run from the repo root on the control machine:

```bash
ansible-playbook ansible/playbooks/cluster-nodes/bootstrap-control-plane.yml
```

The first run takes a couple of minutes while the control-plane images are pulled. A rerun on a healthy control plane changes nothing:

```
cp1 : ok=8  changed=0  unreachable=0  failed=0  skipped=2
```

### How the playbook decides what to do

The playbook checks cp1 before it touches anything:

| State of cp1 | What it finds | What the playbook does |
|---|---|---|
| Clean | no `/etc/kubernetes/admin.conf` | runs `kubeadm init` |
| Healthy | `admin.conf` exists and the API server answers `/readyz` | skips init |
| Half-built | `admin.conf` exists but the API server doesn't answer | stops with a message |

The file alone isn't trusted, because kubeadm writes `admin.conf` early in init, before the control plane is running. A half-built node needs a person to decide between fixing it and resetting it, so the playbook stops with:

```
/etc/kubernetes/admin.conf exists but the API server is not healthy.
Check the control plane by hand, or run kubeadm reset before retrying.
```

After init (or on a healthy node), the playbook also keeps `core`'s kubeconfig in step with `admin.conf`.

### How to verify

As `core` on cp1:

```bash
kubectl get nodes
```

```
NAME   STATUS     ROLES           AGE   VERSION
cp1    NotReady   control-plane   26s   v1.36.6
```

```bash
kubectl get pods -n kube-system -o wide
```

```
NAME                          READY   STATUS    IP
coredns-589f44dc88-krvjq      0/1     Pending   <none>
coredns-589f44dc88-zz52b      0/1     Pending   <none>
etcd-cp1                      1/1     Running   10.10.2.10
kube-apiserver-cp1            1/1     Running   10.10.2.10
kube-controller-manager-cp1   1/1     Running   10.10.2.10
kube-proxy-mnhs7              1/1     Running   10.10.2.10
kube-scheduler-cp1            1/1     Running   10.10.2.10
```

**`NotReady` and `Pending` CoreDNS are correct at this stage.** There's no pod network until Calico is installed. The other pods use cp1's own network (host networking), so they run without it.

The kubeconfig should point at the shared name, not at cp1:

```bash
grep server ~/.kube/config
# server: https://control-plane:6443
```

### How to reset cp1

A reset removes the whole control plane, including all cluster data in etcd.

```bash
# 1. On cp1: undo kubeadm init (asks for confirmation)
sudo kubeadm reset

# 2. From the control machine: keep CRI-O's default CNI configs removed
ansible-playbook ansible/playbooks/cluster-nodes/install-k8s-packages.yml

# 3. Clear the old firewall rules and network interfaces (they only live in memory)
ansible cp1 -m ansible.builtin.reboot

# 4. Remove the old CNI records and core's old kubeconfig
ansible cp1 -m ansible.builtin.file -a "path=/var/lib/cni state=absent"
ansible cp1 -m ansible.builtin.file -a "path=/home/core/.kube state=absent"

# 5. Rebuild
ansible-playbook ansible/playbooks/cluster-nodes/bootstrap-control-plane.yml
```

`kubeadm reset` removes the etcd data, the static pod manifests, the certificates, the kubeconfig files and the kubelet's files. It says itself that it leaves behind the CNI config, the firewall rules and kubeconfig files, which is what steps 2 to 4 cover.

Once workers have joined, they need resetting too. That's covered in the worker join runbook.

### Things to know

- **The Kubernetes version comes from the installed kubeadm.** It's read at run time, so it always matches the packages. `k8s_version` in `group_vars/nodes.yml` is the minor version (`1.36`) and is only used for package names.
- **The API endpoint is a name, not cp1's IP.** The API server certificate lists `control-plane`, `cp1`, `10.10.2.10`, the in-cluster `kubernetes` names and `10.96.0.1`. Clients and workers use `control-plane:6443`. To move the API later, change where `control-plane` points in `/etc/hosts`.
- **Port 6443 is hardcoded.** It's the Kubernetes default.
- **The bootstrap token isn't in the repo.** kubeadm generates one during init. The init output contains the token and the CA cert hash, so don't save it into the repo or the journal. The token expires after 24 hours.
- **cp1 keeps the control-plane taint** (`node-role.kubernetes.io/control-plane:NoSchedule`), so ordinary pods won't run on it.
- **kubectl from the control machine isn't set up yet.** That's the cluster access ticket. The control machine will also need `control-plane` in its `/etc/hosts`.
- **The containers-common copy is a temporary workaround** for CRI-O 1.36 on Fedora 45. Remove it when moving to CRI-O 1.37.
- **Init shows a pause image warning** (CRI-O uses `pause:3.10.1`, kubeadm suggests `3.10.2`). It doesn't block init. It's tracked in the backlog.

### Problems hit during setup

**Image pulls failed in preflight.** `kubeadm init` stopped with `no policy.json file found at any of the following: "/root/.config/containers/policy.json", "/etc/containers/policy.json"`. Fedora 45's containers-common moved its defaults to `/usr/share/containers/`, but CRI-O 1.36 only looks in `/etc/containers/`. Fixed by copying `policy.json`, `registries.conf` and `registries.conf.d/` to `/etc/containers/` in `install-k8s-packages.yml`.

**Node Ready with no pod network.** After the first init, cp1 was `Ready` and CoreDNS got `10.85.0.x` addresses. The cri-o package ships a test bridge network (`100-crio-bridge.conflist`) in `/etc/cni/net.d/`, and CRI-O used it. After removing it, cp1 was still `Ready`, and CoreDNS got `127.0.0.1` and kept restarting, because CRI-O then used `200-loopback.conflist` as the network. Fixed by removing all three default CNI files in `install-k8s-packages.yml`. Now cp1 is `NotReady` until Calico is installed, as it should be.

Lesson: if pods get `10.85.x.x` or `127.0.0.1` addresses, check `/etc/cni/net.d/`.

**Package install looked for `kubernetes1.36.6`.** Setting `k8s_version` to the full version broke `install-k8s-packages.yml`, which builds package names from it (`kubernetes1.36`). `--check` didn't catch it, because Ansible skips `command` tasks in check mode. Fixed by keeping `k8s_version` as the minor version and reading the full version from kubeadm at run time.

### References

- [kubeadm reset](https://k8s.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/): what reset removes and what it leaves behind
- [CRI-O CNI README](https://github.com/cri-o/cri-o/blob/main/contrib/cni/README.md): how CRI-O picks a CNI config file from `/etc/cni/net.d/`
- [Kubernetes on Fedora](https://computingforgeeks.com/kubernetes-on-fedora/): `policy.json` error on Fedora 45 with CRI-O 1.36, and copying the containers-common defaults to `/etc/containers/`