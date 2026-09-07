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
- **Hetzner VPS** — a live multi-sensor honeypot stack (Cowrie SSH/Telnet, Dionaea multi-protocol, a purpose-built WordPress HTTP honeypot, mailoney SMTP) capturing real attacker traffic, with Telegram alerting
- **Cisco SG300-10** — managed switch running in **Layer 3 (Router) mode**, performing inter-VLAN routing between a management VLAN, a production VLAN, and an isolated lab VLAN. Routing verified end to end from a client; ACL policy is the next phase.
- **Backups** — nightly local snapshots plus offsite copies to Cloudflare R2

## Repo structure

- `hardware/` — physical machine specs and purchase records
- `services/` — per-VM and per-service reference docs (current state, not history)
- `runbooks/` — session-by-session build logs, written to capture not just *what* was done but *why*, including mistakes and how they were diagnosed

## Why this repo exists

Every session here is documented as if someone else — a hiring manager, a future me — needs to understand the reasoning, not just the commands. The goal is to show real troubleshooting, real incident analysis (see the honeypot logs), and real infrastructure decisions, not a polished after-the-fact summary.

## Status

Actively worked, roughly session-by-session. See `runbooks/` for the most recent entries.

**Current phase:** network segmentation. The switch routes between VLANs; the
next step is writing isolation as explicit ACL policy rather than relying on
the absence of a route. VLANs are broadcast separation — they are not a
security boundary on their own.
