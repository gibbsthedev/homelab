# VM: docker-agents (VMID 100)

## Purpose
Hosts Docker + migrated agents (OpenClaw) from AWS.

## Specs
- OS: Debian 13.6 (Trixie)
- vCPU: 2 (type: host)
- RAM: 4GB (4096 MiB)
- Disk: 32 GB on local-lvm (NVMe-backed)
- IP: 192.168.8.139 (DHCP from router)
- Qemu guest agent: installed and running

## Accounts
- root -- password in apple
- rich -- regular user, in 'sudo' and 'docker' groups
  - password in apple
- guni -- she has password

## Software installated
- Docker CE (via official Docker apt repo) -- verified with hello-world
 - Docker enabled on boot
 - user rich can run docker without sudo
- Node.js v22 (for OpenClaw) [confirm after install]
- rsync, curl, git, qemu-guest-agent

## Access 
- SSH: ssh rich@192.168.8.139 (from any device on Beryl AX network)
- Console: via Proxmox web UI (fallback only)
