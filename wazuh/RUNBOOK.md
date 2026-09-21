# Wazuh Deployment Runbook (all-in-one VM)

Draft for manual execution. Nothing here has been run. Replace every `<PLACEHOLDER>`.

**Target:** Ubuntu 24.04 VM, 4 vCPU / 8 GB RAM / 100 GB disk, on the SOC-lab VLAN.
Manager, Indexer and Dashboard all on one VM (assisted installer).

## 0. Placement decision (do this first)

`lab-pve-01` has ~6 GiB free of 31 GiB. An 8 GB VM will only fit if ZFS ARC gives memory back.

```bash
# on lab-pve-01
free -h
arc_summary | grep -E "ARC size|Target size|Max size"   # or: cat /proc/spl/kstat/zfs/arcstats | grep -E "^(size|c_max) "
pvesm status                                             # free space per storage pool
qm list; pct list                                        # allocated RAM per guest
```

- If `available` in `free -h` is >= ~9 GB with ARC counted as reclaimable, and the target pool has >= 100 GB free, use lab-pve-01.
- Otherwise, put Wazuh on the new 16 GB node once it has joined the cluster and the SOC-lab VLAN is trunked to it. Wazuh is the heaviest and first-needed service, so it is the best fit for that node.
- Do not deploy TheHive/Cortex/MISP until Wazuh is verified (section 5).

## 1. Create the VM (Proxmox shell)

```bash
VMID=120                       # any free ID
STORAGE=<TARGET_POOL>          # from pvesm status, e.g. local-lvm
BRIDGE=vmbr0
VLAN=<SOC_LAB_VLAN_ID>
ISO=<STORAGE>:iso/ubuntu-24.04-live-server-amd64.iso   # upload first if missing

qm create $VMID --name wazuh-01 --ostype l26 --machine q35 --bios ovmf \
  --cores 4 --memory 8192 --cpu host --agent 1 \
  --scsihw virtio-scsi-single --scsi0 $STORAGE:100,discard=on,iothread=1 \
  --efidisk0 $STORAGE:1,efitype=4m,pre-enrolled-keys=0 \
  --net0 virtio,bridge=$BRIDGE,tag=$VLAN \
  --ide2 $ISO,media=cdrom --boot order='ide2;scsi0' --onboot 1
qm start $VMID
```

Install Ubuntu from the console: static IP on the SOC-lab VLAN (`<WAZUH_IP>/<PREFIX>`, gateway, DNS), enable OpenSSH, then:

```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install qemu-guest-agent curl
sudo systemctl enable --now qemu-guest-agent
# Indexer needs this; the installer usually sets it, verify after install:
sudo sysctl -w vm.max_map_count=262144
```

Take a Proxmox snapshot here (`qm snapshot $VMID pre-wazuh`) so you can roll back.

## 2. Install Wazuh (assisted installer)

Check the current release at https://documentation.wazuh.com/current/quickstart.html and use its exact URL.

```bash
# on wazuh-01
curl -sO https://packages.wazuh.com/<VERSION>/wazuh-install.sh
sudo bash ./wazuh-install.sh -a 2>&1 | tee ~/wazuh-install.log
```

The installer prints the `admin` password once at the end. Store it in Vaultwarden immediately, then remove `~/wazuh-install.log`. It also writes `wazuh-install-files.tar` (keep it in a safe place; needed for later changes).

## 3. Firewall / VLAN rules

Allow into `<WAZUH_IP>` from the SOC-lab VLAN only (plus your admin workstation for 443/22):

| Port | Proto | Purpose |
| --- | --- | --- |
| 1514 | TCP | Agent event traffic |
| 1515 | TCP | Agent enrollment |
| 55000 | TCP | Manager API (admin only) |
| 443 | TCP | Dashboard (admin only; front with Nginx Proxy Manager later) |
| 22 | TCP | SSH (admin only) |

Keep 9200 (Indexer) closed to everything but localhost. Kali should be able to reach the endpoints, but nothing on the general Lab VLAN should reach the SOC VLAN except your admin host.

## 4. Enroll endpoints

Dashboard: **Agents > Deploy new agent** generates the exact command; the shape is below.

Linux client (Ubuntu/Debian):
```bash
WAZUH_MANAGER='<WAZUH_IP>' WAZUH_AGENT_GROUP='linux' apt-get install -y wazuh-agent   # after adding the Wazuh apt repo per the dashboard
sudo systemctl enable --now wazuh-agent
sudo apt -y install auditd && sudo systemctl enable --now auditd
```

Windows client (elevated PowerShell):
```powershell
msiexec.exe /i wazuh-agent-<VERSION>.msi /q WAZUH_MANAGER='<WAZUH_IP>' WAZUH_AGENT_GROUP='windows'
NET START WazuhSvc
```
Then install Sysmon with a maintained config (e.g. SwiftOnSecurity) and add the `Microsoft-Windows-Sysmon/Operational` eventchannel `<localfile>` to the agent `ossec.conf`. FIM paths go in `<syscheck>` (Linux: `/etc`, `/usr/bin`; Windows: `C:\Users\*\Desktop`, startup folders).

Create the `linux` and `windows` groups under **Agents management > Groups** first so shared config is centralized.

## 5. Verify

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
sudo /var/ossec/bin/agent_control -l          # agents listed Active
curl -k -u admin https://<WAZUH_IP>:9200      # from the VM only; returns cluster JSON
```

- Dashboard at `https://<WAZUH_IP>` shows both agents Active.
- Trigger a test alert: fail SSH login 5+ times to the Linux client; confirm rule 5710/5712 events appear under **Threat Hunting**.
- Edit a file in a FIM-monitored path; confirm a syscheck event.

## 6. Rollback

`qm stop $VMID && qm rollback $VMID pre-wazuh`, or `qm destroy $VMID --purge` to start over.

## Next in order

Endpoints hardening/config, then the Windows Server 2022 DC (audit policy for events 4662/4769), then TheHive + Cortex, MISP, Shuffle.
