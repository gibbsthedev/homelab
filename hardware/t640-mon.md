# t640-mon — HP t640 Thin Client (monitoring box)

Status: **Built 2026-10-29.** Debian 13, hardened, reboot-tested. Monitoring stack not yet installed.

## Role
Internal monitoring host: LibreNMS (SNMP polling of the SG300 and OPNsense), central syslog,
later NetFlow / Zeek. Sits inside the network — no ports exposed to the internet.

Roles were swapped with the t655 on 2026-10-29. This box got monitoring because its real
SATA SSD handles constant log/graph writes far better than the t655's eMMC-style flash,
and its outdated-then-updated BIOS matters less on an internal box than at the edge.

## Hardware
| Item | Value |
| --- | --- |
| Model | HP t640 Thin Client |
| Product no. / SKU | xxxx (t640/NoOS/128GF/4GRW TC) |
| Serial | xxxx |
| CPU | AMD Ryzen Embedded R1505G — 2 cores / 4 threads |
| RAM | 4 GB DDR4 SO-DIMM (2 slots, 32 GB max) |
| Storage | SSSTC CVB-8D128-HP — 128 GB M.2 **SATA** SSD (`sda`), standard and swappable |
| Wired NIC | Realtek RTL8111/8168 `[10ec:8168]` → `enp1s0f0` |
| Wi-Fi | Intel AX200 (unused) |
| LAN MAC | xx:xx:xx:xx:xx:xx |
| Power | HP 19.5 V / 2.31 A barrel brick |

## Network
| Item | Value |
| --- | --- |
| Switch port | SG300 **gi6**, VLAN 1 (default access) |
| Address | 192.168.8.24/24, DHCP from Beryl, reserved to the MAC above |
| Hostname | `t640-mon` (domain `lan`) |
| Access | `ssh t640-mon` from Mint (key `~/.ssh/homelab_mint`, `IdentitiesOnly yes`) |

## OS
- Debian 13 (trixie), netinst 13.7.0, kernel 6.12.113+deb13-amd64
- No desktop; SSH server + standard system utilities only
- Root login disabled (installer root password left blank), user `scgiky` has sudo
- Apt: deb.debian.org (trixie, trixie-updates) + security.debian.org
- unattended-upgrades enabled
- `/etc/sysctl.d/10-quiet-console.conf`: `kernel.printk = 3 4 1 3`
- `/etc/ssh/sshd_config.d/10-hardening.conf`: `PasswordAuthentication no`, `PermitRootLogin no`
  (tested with a second session before closing the first)

## BIOS
| Item | Value |
| --- | --- |
| Before | AMI M43 v01.09 (06/09/2021) |
| After | M43 v01.24 — SoftPaq sp173784 (2026-10-29) |
| BIOS Update setting | Returned to **Disable** after flashing |

### How the BIOS was updated (no Windows on the t640)
1. HP's supported flash tools need Windows **on the thin client itself**. Running
   `M43_0124.exe` on the Windows laptop tried to load `amifldrv64.sys`, which Windows'
   vulnerable-driver blocklist blocked. **The blocklist was left ON** — wrong machine, and
   disabling a security control globally to fix a local problem is never the answer.
2. F10 → File → **Flash System BIOS** with hand-built USB layouts failed ("file missing or damaged").
3. Read HP's ToolLess procedure and translated each Windows step to Linux:
   - `mountvol x: /S` + `xcopy ...\ToolLess\HP x:\EFI\HP` → `cp -r HP /boot/efi/EFI/`
   - `BiosConfigUtility64.exe /set:EnableToollessUpdate.txt` (sets `BIOS Update = *Auto`) →
     F10 → File → **BIOS Config Utility → Save Current Setting to File in ESP**, which writes
     `/boot/efi/EFI/HP/UefiBCU/hpisetup.txt`.
4. That file is **UTF-16LE with a BOM (`ff fe`)** — plain `grep` sees nothing. Converted with
   `iconv`, moved the `*` from `Disable` to `Auto` under `BIOS Update`, converted back, re-added the BOM.
5. F10 → BIOS Config Utility → **Restore Former Setting from File in ESP** → Save Changes and Exit.
   On reboot, HP's own flasher ran from `EFI\HP` (4096 blocks).
6. Same edit in reverse to set `BIOS Update` back to `Disable`.

**Gotcha:** after the flash, *neither* the old Debian install nor the Debian installer would
boot — the kernel hung right after `EFI stub: Measured initrd data into PCR 9`.
`nomodeset` and removing `vga=788` did not help. **F10 → File → Apply Defaults and Exit**
fixed it. Lesson: after an HP BIOS flash, load defaults before blaming the OS.

## Known harmless noise
`ACPI BIOS Error (bug): AE_AML_BUFFER_LIMIT ... \_SB.WMID.WQBE` at boot — bug in HP's WMI
ACPI tables (hotkey/BIOS-settings interface). Logged, ignored; hidden from the console via printk.

## Next
- [x] Beryl DHCP reservation 192.168.8.24 ↔ xx:xx:xx:xx:xx:xx (2026-10-29)
- [ ] LibreNMS + SNMP on the SG300
- [ ] rsyslog receiver for SG300 / OPNsense logs (with log rotation from day one)
