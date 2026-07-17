## 2026-07-15 - Flash Drive Patience
- Remember to wait until the flash drive finishes writing before pulling and use umount command to safely remove. 

## 2026-07-15 - Apartment network needs investigation
- M720q plugged into apartment ethernet wall jack. 
- Internet is SHARED WiFi
- Linux Mint Laptop Wifi: 100.110.148.128/26, gw 100.110.148.129

## 2026-07-16 - Foundation Complete
- GL.iNet Beryl AX connected to apt WiFi as uplink
- M720q reached at static IP: 192.168.8.2:8006
- Fixed subscription popup 
- Spent a lot of time fixing errors for apt upgrade, learned about 'cat <<' and 'EOF' multi-line is hard when you cannot copy paste
- Installed git on server

## 2026-07-17 -- Docker host built + agent migration started
- Created VM 'docker-agents' (VMID 100), Debian 13.6, on pve-node1. 
- Installed Docker CE successfully (hello-world verified).
- GOTCHA: Docker apt repo must be added BEFORE 'apt update', or packages show "no installation candidate." The long echo|tree command can fail on partial paste -- verify with 'cat docker.list'.
- Minimal Debian lacks common tools (sudo, rsync, curl) -- install as needed.
- User perms: 'usermod -aG docker rich' lets rich run docker w/o root. (but it takes effect on the next login). Enabling services needs root/sudo, not plain user. 
- Migration approach: AWS is cloud (public), home VM is behind NAT on apartment WiFi. AWS can't reach IN, but VM can reach OUT -- so I PULL data from the VM via rsync, not push from AWS.
- Transferred ~/.openclaw (6.3GB) from AWS (54.197.26.209) to VM via rsync over SSH, using the AWS .pem key (copied to VM, chmod 600).
