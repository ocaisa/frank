# EESSI/Slurm cluster for Ubuntu 24.04 ARM64

Ansible project that deploys a 5-node Slurm cluster:

- `head01`: Raspberry Pi 5 (16 GB) — Slurm controller, NFS server, DHCP, squid proxy, CVMFS
- `worker01`–`worker03`: Raspberry Pi 5 (8 GB) — Slurm compute nodes, CVMFS clients
- `worker04`: Jetson Orin Nano (8 GB, GPU) — Slurm compute node with explicit `gpu:1` GRES

The head has internet access over Wi-Fi and provides NAT, DHCP, and a squid proxy to the private `10.0.0.0/24` LAN.

## Requirements

- 5 Ubuntu 24.04 ARM64 nodes.
- SSH access as the `frank` user on all nodes.
- The `frank` user must have sudo access and a passwordless SSH key installed.
- Head wired LAN interface: `eth0` (private cluster network).
- Head Wi-Fi/uplink interface: expected to be `wlan0` (update `uplink_iface` if different).
- Worker LAN MAC addresses filled in under `inventory/host_vars/worker*.yml`.
- Ansible on the control node (for example `sudo apt install ansible`).

## Important defaults

| Item | Value |
|---|---|
| Cluster network | `10.0.0.0/24` |
| Head static IP | `10.0.0.1/24` |
| Worker IPs | `10.0.0.11`–`10.0.0.14` |
| DHCP | head `eth0` only, static MAC reservations |
| Squid proxy | head listens on `127.0.0.1:3128` and `10.0.0.1:3128` |
| NFS exports | `/nfs` (rw pseudo-root, `fsid=0`); eessi home at `/nfs/home` |
| EESSI CVMFS | `/cvmfs/software.eessi.io` |
| EESSI user | `eessi` / `EESSI` |
| Slurm state dir | `/var/spool/slurm` (per-node, local — not NFS) |
| Slurm GPU worker | `worker04`, explicit `Gres=gpu:1`, no GPU autodetection |

## First-time preparation

1. Copy your SSH public key to `frank@head01`, `frank@worker01`, etc.
2. Replace the placeholder worker MACs in:
   - `inventory/host_vars/worker01.yml`
   - `inventory/host_vars/worker02.yml`
   - `inventory/host_vars/worker03.yml`
   - `inventory/host_vars/worker04.yml`
3. Confirm that the head’s Wi-Fi interface is `wlan0`; if it is not, update `uplink_iface` in `inventory/hosts.yml`.
4. Confirm that the head’s private LAN interface is `eth0`; if it is not, update `head_lan_iface` in `inventory/hosts.yml`.

## Running the playbook

Run from the project root.

### First run: head node only

Before the head’s LAN IP (`10.0.0.1`) has been assigned, configure only the head node, which you run the playbook on. The inventory is already set up for this: `head01`’s `ansible_host` is `localhost` in `inventory/hosts.yml`.

```bash
ansible-playbook site.yml --limit head_node
```

### After the IP is assigned: implement the TODO and run the rest

1. In `inventory/hosts.yml`, change `head01`’s `ansible_host` from `localhost` back to `10.0.0.1` (see the `TODO` comment on that line).
2. Run the remaining plays:

   ```bash
   ansible-playbook site.yml --limit worker_nodes
   ```

   or re-run everything:

   ```bash
   ansible-playbook site.yml
   ```

The playbook runs the head node first, then the workers. The head’s squid proxy is installed before CVMFS so that CVMFS can use it immediately.

## EESSI usage

After a successful run, log in as `eessi`:

```bash
ssh eessi@head01
```

Default password:

```text
EESSI
```

The `eessi` shell profile automatically initialises EESSI (lmod) when available:

```bash
source /cvmfs/software.eessi.io/versions/2026.06/init/lmod/bash
```

Proxy environment is configured on all nodes:

- head: `http://127.0.0.1:3128` (local listener)
- workers: `http://10.0.0.1:3128` (head's LAN listener)

The `eessi` user's `.bashrc` lives on the shared NFS home, so it uses the head's LAN address on every node — a per-node (loopback) address would be wrong on the workers.

APT on workers is also configured to use the squid proxy. APT on the head is intentionally not proxied during the initial bootstrap because squid is not installed yet at that point.

## Useful tests

```bash
# Slurm
scontrol ping
sinfo
srun -N1 -n1 hostname
srun -N1 -w worker04 --gres=gpu:1 hostname

# clush
clush -w all hostname

# CVMFS
ls /cvmfs/software.eessi.io

# proxy from a worker
curl -x http://10.0.0.1:3128 https://example.com
```

For the Jetson GPU, `nvidia-smi` works and Slurm GPU autodetection is enabled via NVML.

```bash
ssh eessi@worker04
nvidia-smi
```

## Troubleshooting

### DHCP does not assign the expected worker IPs

Check that the real worker MAC addresses are set in `inventory/host_vars/worker*.yml`. The DHCP server only hands out static reservations for those MACs.

### Munge/Slurm authentication errors

Check the Munge key and service:

```bash
sudo munge -n
echo test | munge | unmunge
```

The canonical Munge key is published to `keys/munge.key` on the control node during the first run. Do not commit this file.

### Slurm daemon problems

Check logs:

```bash
sudo journalctl -u slurmctld
sudo journalctl -u slurmd
```

### CVMFS problems

Check the repository service:

```bash
sudo systemctl status cvmfs-software.eessi.io.service
```

### Proxy problems

Check squid:

```bash
sudo systemctl status squid
sudo tail -n 100 /var/log/squid/cache.log
```

## Security notes

- `eessi_password` is stored in plaintext in the inventory for bootstrap convenience.
- `keys/munge.key` is generated during the first run and should be kept private.
- This project assumes a trusted private LAN.

### Head node firewall (ufw)

The head's firewall is managed by the `network` role:

- Default **incoming** policy: `deny`.
- Default **routed (forward)** policy: `allow` — required because the head is the LAN's NAT gateway; without it the workers lose internet access.
- Allowed inbound:
  - SSH (22/tcp) from the trusted home Wi-Fi network (`admin_wifi_net`, default `192.168.1.0/24`).
  - **Everything** arriving on the wired LAN interface (`eth0`) — this covers all cluster services (NFS, slurmctld, munge, squid, DHCP) and worker SSH, and is interface-scoped so DHCP (`DHCPDISCOVER` from `0.0.0.0`) is included.

The allow rules are applied before the deny policy and before `ufw enable`, so re-running the playbook can never lock out the active SSH session. Inspect with:

```bash
sudo ufw status verbose
```

