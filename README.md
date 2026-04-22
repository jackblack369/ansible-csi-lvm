# ansible-csi-lvm

Ansible pipeline to prepare Linux nodes for `csi-lvm-driver`: installs nvme-cli
+ kernel modules, configures `sanlock` / `lvmlockd` on a shared VG
(`csi-lvm`), deploys `nvmeof-connect` systemd unit gated by an IB health
check, then labels the K8s nodes so `csi-lvm-driver` schedules onto them.

## Inventory

Put target nodes in group `csi_lvm_nodes` (see `inventory/hosts.ini`). All
vars in `inventory/group_vars/all.yml`.

## Common commands

```bash
# Full deploy
ansible-playbook playbooks/csi_lvm_site.yml

# Single phase
ansible-playbook playbooks/csi_lvm_01_nvme.yml      # nvme-cli + modules
ansible-playbook playbooks/csi_lvm_02_lockd.yml     # sanlock + lvmlockd
ansible-playbook playbooks/csi_lvm_03_nvmeof.yml    # nvmeof systemd unit
ansible-playbook playbooks/csi_lvm_04_label.yml     # kubectl label

# Dry run / syntax / ping
ansible-playbook playbooks/csi_lvm_site.yml --check
ansible-playbook playbooks/csi_lvm_site.yml --syntax-check
ansible csi_lvm_nodes -m ping

# Status
ansible-playbook playbooks/csi_lvm_99_status.yml
```

## Prerequisites on control node

- `kubectl` + KUBECONFIG with permission to label nodes (phase 4 runs via
  `delegate_to: localhost`)
- `check-ib-health.sh` present at `nvmeof_check_ib_health_local_src`
  (default `/root/csi-lvm/script/check-ib-health.sh`) — phase 3 fails fast
  if missing

## Prerequisites on target nodes

- Debian / Ubuntu (`apt`)
- root SSH access
- IB interface named per `nvmeof_ib_network` (default `ib6s200p0`)
- shared VG `csi-lvm` pre-created on the nvmeof-backed PV

## Key variables (override in `group_vars/all.yml` or host_vars)

| var | default | meaning |
|---|---|---|
| `nvmeof_traddr` | `192.168.0.106` | NVMe-oF target address |
| `nvmeof_trsvcid` | `4420` | target service id (port) |
| `nvmeof_nqn` | `nqn.xxx` | target NQN |
| `nvmeof_ib_network` | `ib6s200p0` | IB iface checked before connect |
| `csi_lvm_vg_name` | `csi-lvm` | shared VG name (lockspace) |
| `lvm_host_id` | last octet of `ansible_default_ipv4` | sanlock host_id |
| `csi_lvm_label_key` | `dc.com/service.csi-lvm` | k8s label key |
| `csi_lvm_label_value` | `enable` | k8s label value |
| `k8s_node_name` | `inventory_hostname` | override if k8s node != inventory name |
