# VM: docker-agents / "Tuck" (VMID 100)

## Purpose
Hosts Docker + migrated agents (OpenClaw) from AWS. Runs a 12-agent AI system.

## Specs (current — updated 2026-08)
- OS: Debian 13.6 (Trixie)
- vCPU: 3 (type: host)
- RAM: 6 GB (6144 MiB) — resized down from original 8GB/4-core to free RAM headroom for the Hermes VM
- Disk: 32 GB on local-lvm (NVMe-backed), plus a second 16 GB disk (scsi1, backup=0) for OpenClaw's own internal backups
- IP: **192.168.8.10 (static, MAC reservation)** — was DHCP-assigned 192.168.8.139, moved to a static reservation for reliability
- Qemu guest agent: installed and running
- Gateway service runs on port 18789

## Accounts
- root — password in Apple Passwords ("Homelab - docker-agents root (192.168.8.10)")

## Change log
- 2026-07-17: VM created, Docker CE installed, agents migrated from AWS via rsync pull
- 2026-08-21: Moved to static IP 192.168.8.10 (MAC reservation), resized 4 vCPU/8GB → 3 vCPU/6GB to make room for the new Hermes VM on the same host — decision made after measuring actual RAM usage (well under allocation) rather than guessing at need
