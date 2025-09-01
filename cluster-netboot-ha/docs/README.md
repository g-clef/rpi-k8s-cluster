# Cluster Netboot HA (Synology PXE VM + Pi control plane)

## Quick start
1) Create Synology shared folder `nfsroot` with NFS enabled (No mapping).
2) On the Synology VM (Debian/Ubuntu):
   ```bash
   sudo apt update && sudo apt install -y ansible
   ansible-playbook -i inventory/hosts.yaml site.yaml -K --limit nas-vm
   ```
3) Ensure Pi EEPROMs are set to Network Boot.
4) Power on Pis with no SD: pi01 initializes cluster; pi02/pi03 join; others become agents.
5) GPU node can be powered on anytime; see `gpu-node/README.md`.

## RAM overlays
`roles/nfs_roots` injects tmpfs mounts for `/var/log` and `/tmp` into each Pi root to avoid constant NFS writes.
