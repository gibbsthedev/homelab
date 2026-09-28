# Homelab

A homelab built from scratch, documented as a learning log and a security portfolio — working toward a SOC analyst role.

## Goals

1. **Cut cloud costs** — migrating workloads off AWS onto owned hardware.
2. **Own the infrastructure** — full control, top to bottom, instead of renting it.
3. **Build a security portfolio** — hands-on practice toward CompTIA Network+ (N10-009) and a SOC analyst career, including live honeypot operations against real internet attackers.

## Network map

```mermaid
flowchart TB
    internet(("Internet"))
    isp["WhiteSky — apartment ISP<br/>wired wall jack (primary)<br/>WhiteSky-Prose WiFi (failover)"]
    beryl["GL.iNet Beryl AX — edge router<br/>NAT · DHCP · WiFi (FBI #4 / #5)<br/>192.168.8.1"]
    sw["Cisco SG300-10 — Layer 3 switch<br/>routes between VLANs"]

    internet --- isp --- beryl
    beryl ---|"gi1"| sw

    subgraph v1["VLAN 1 — management · 192.168.8.0/24"]
        mint["Mint — admin workstation<br/>gi5 (cable) + FBI WiFi"]
        svi1["Switch mgmt · .8.165"]
    end

    subgraph v10["VLAN 10 — production · 192.168.10.0/24"]
        pve["pve-node1 — Proxmox VE<br/>Lenovo M720q · .10.2"]
        vm100["VM 100 — Tuck / OpenClaw<br/>.10.10"]
        vm101["VM 101 — Hermes<br/>.10.206"]
        pve --- vm100
        pve --- vm101
    end

    subgraph v20["VLAN 20 — isolated lab · 192.168.20.0/24"]
        lab["Lab / test port<br/>gi3"]
    end

    sw ---|"gi5"| mint
    sw ---|"gi2"| pve
    sw ---|"gi3"| lab
```

The Beryl carries static return routes for VLAN 10 and VLAN 20 pointing at the
switch (`192.168.8.165`). Without them, traffic from those VLANs reaches the
internet but the replies never find their way back — asymmetric routing.

## Segmentation

The switch routes between VLANs, so isolation is enforced with explicit ACLs
rather than by leaving routes out. VLANs separate broadcast traffic; they are
not a security boundary on their own.

```mermaid
flowchart LR
    v1["VLAN 1<br/>management"]
    v10["VLAN 10<br/>production"]
    v20["VLAN 20<br/>lab"]
    mgmt["Switch mgmt IP"]
    net(("Internet"))

    v20 --x|"denied · vlan20-policy"| v10
    v20 --x|"denied · vlan20-policy"| v1
    v20 -->|"allowed"| net

    v10 --x|"denied · vlan10-policy"| v20
    v10 --x|"denied · vlan10-policy"| mgmt
    v10 -->|"allowed"| net

    v1 -->|"allowed — accepted risk"| v10
```

| ACL | Bound to | What it enforces |
|---|---|---|
| `vlan20-policy` | gi3, input | Lab can reach the internet only. Denied: production, the switch's management IP, the rest of VLAN 1. |
| `vlan10-policy` | gi2, input | Production can reach the internet and VLAN 1. Denied: the lab and the switch's management IP. |

Both were verified by function — ping and connection tests from a client in the
VLAN, not just by reading the config — and the config was confirmed to survive
a cold reload of the switch.

**How the ACLs work on this switch.** The SG300's 2011 firmware only binds ACLs
to physical ports, and only in the `input` direction. Each ACL therefore judges
traffic *leaving* its VLAN. The filtering is stateless — the switch has no
memory of connections — so rules are written by matching source and
destination addresses, the same model as an AWS Network ACL.

**Known gap, accepted for now.** Devices on the Beryl's WiFi (VLAN 1) can reach
production, including the Proxmox web UI. Nothing filters traffic *entering*
the switch from the Beryl (gi1). This was found by testing from a phone. The
fix has been dry-run on paper: an ACL on gi1, plus a host route so the admin
workstation isn't locked out. It's deferred because only admin devices use that
WiFi today. Proxmox still requires authentication.

## Current infrastructure

- **pve-node1** — Lenovo ThinkCentre M720q, running Proxmox VE 9.2 on VLAN 10, hosting two VMs:
  - **Tuck / OpenClaw** — a 12-agent AI system, migrated off AWS
  - **Hermes** — an AI gateway fleet, cut over from AWS EC2, RHEL 10
- **Cisco SG300-10** — managed switch in **Layer 3 (Router) mode**: inter-VLAN routing between management, production, and an isolated lab VLAN, with ACL-enforced segmentation (see above)
- **GL.iNet Beryl AX** — edge router on a wired WAN, with automatic WiFi failover tested by pulling the cable (zero packets lost)
- **Hetzner VPS** — a live multi-sensor honeypot stack (Cowrie SSH/Telnet, Dionaea multi-protocol, a purpose-built WordPress HTTP honeypot, mailoney SMTP) capturing real attacker traffic, with Telegram alerting
- **Backups** — nightly local snapshots plus offsite copies to Cloudflare R2

## Repo structure

- `hardware/` — physical machine specs and purchase records
- `services/` — per-VM and per-service reference docs (current state, not history)
- `runbooks/` — session-by-session build logs, written to capture not just *what* was done but *why*, including mistakes and how they were diagnosed

## Why this repo exists

Every session here is documented as if someone else — a hiring manager, a future me — needs to understand the reasoning, not just the commands. The goal is to show real troubleshooting, real incident analysis (see the honeypot logs), and real infrastructure decisions, not a polished after-the-fact summary.

## Status

Actively worked, roughly session-by-session. See `runbooks/` for the most recent entries.

**Current phase:** network segmentation is in place and verified for traffic
leaving the production and lab VLANs. Next up: closing the inbound gap from
the WiFi side, hardening unused switch ports, and an 802.1Q trunk to Proxmox
so lab VMs can live on the same host as production in their own VLAN.
