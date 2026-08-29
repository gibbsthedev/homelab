[README.md](https://github.com/user-attachments/files/31590052/README.md)
# Homelab

A homelab built from scratch, documented as a learning log and a security portfolio — working toward a SOC analyst role.

## Goals

1. **Cut cloud costs** — migrating workloads off AWS onto owned hardware.
2. **Own the infrastructure** — full control, top to bottom, instead of renting it.
3. **Build a security portfolio** — hands-on practice toward CompTIA Network+ (N10-009) and a SOC analyst career, including live honeypot operations against real internet attackers.

## Current infrastructure

- **pve-node1** — Lenovo ThinkCentre M720q, running Proxmox VE 9.2, hosting two VMs:
  - **Tuck / OpenClaw** — a 12-agent AI system, migrated off AWS
  - **Hermes** — an AI gateway fleet, cut over from AWS EC2, RHEL 10
- **Hetzner VPS** — a live honeypot stack (Cowrie SSH/Telnet, mailoney SMTP) capturing real attacker traffic, with Telegram alerting
- **Cisco SG300-10** — managed switch, staged for VLAN segmentation as the next infrastructure phase
- **Backups** — nightly local snapshots plus offsite copies to Cloudflare R2

## Repo structure

- `hardware/` — physical machine specs and purchase records
- `services/` — per-VM and per-service reference docs (current state, not history)
- `runbooks/` — session-by-session build logs, written to capture not just *what* was done but *why*, including mistakes and how they were diagnosed

## Why this repo exists

Every session here is documented as if someone else — a hiring manager, a future me — needs to understand the reasoning, not just the commands. The goal is to show real troubleshooting, real incident analysis (see the honeypot logs), and real infrastructure decisions, not a polished after-the-fact summary.

## Status

Actively worked, roughly session-by-session. See `runbooks/` for the most recent entries.
