# SOC Homelab Infrastructure Documentation

This file documents the steps and configuration choices for the SOC Homelab virtual machines and network setup.

---

## 1. VirtualBox and Proxmox Setup

- Installed VirtualBox on Arch Linux host.
- Created Debian-based VM for Proxmox:
  - CPU: 4 cores
  - RAM: 16 GB
  - Disk: 50 GB
  - Network: Bridged mode
- Enabled nested virtualization:
```bash
VBoxManage modifyvm "Proxmox" --nested-hw-virt on
```
- Installed Proxmox VE.
- Assigned static IP to Proxmox: 192.168.1.16/24, gateway: 192.168.1.1.
- Verified connectivity from host using ping.

---

## 2. SSH Configuration

- Enabled SSH:
```bash
systemctl enable --now ssh
```
- Configured `/etc/ssh/sshd_config`:
  - PermitRootLogin yes
  - PasswordAuthentication yes
- Reset root password to ensure successful login from host.

---

## 3. Proxmox Network Bridges

| Bridge | Purpose           | Notes                       |
|--------|-----------------|-----------------------------|
| vmbr0  | WAN             | IP assigned, used for management |
| vmbr1  | DMZ             | Layer 2 only               |
| vmbr2  | Intermediate    | Layer 2 only               |
| vmbr3  | Active Directory| Layer 2 only               |
| vmbr4  | SOC             | Layer 2 only               |

- Only vmbr0 has an IP and gateway.
- Internal bridges act as L2 switches; VMs get their own IPs.

---

## 4. FortiGate VM Deployment

- Downloaded FortiGate QCOW2 image.
- Copied to Proxmox:
```bash
scp ~/Downloads/fortios.qcow2 root@192.168.1.16:/var/lib/vz/template/qcow2/
```
- Created VM in Proxmox with ID 101.
- Imported QCOW2 disk:
```bash
qm importdisk 101 fortios.qcow2 local-lvm
qm set 101 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-101-disk-0
qm set 101 --boot c --bootdisk scsi0
```
- Fixed boot issue "starting serial terminal on interface serial0":
```bash
qm set 101 --serial0 none
qm set 101 --vga std
```
- VM started successfully via Proxmox console.

---

## 5. FortiGate Network Configuration

- Network interfaces assigned to bridges:
  - net0 → vmbr0 (WAN)
  - net1 → vmbr1 (DMZ)
- Initial IP configuration via FortiGate console:
  - port1 → 192.168.14.1/24, allow ping, HTTPS, SSH
  - port2 → DHCP if needed for WAN
- Web GUI accessible at https://192.168.14.1

Documentation used for the installation at https://docs.fortinet.com/document/fortigate-private-cloud/7.6.0/proxmox-administration-guide/37920/deploying-a-fortigate-vm-into-proxmox


---

## 6. Notes and Decisions

- VirtualBox in bridged mode allows host access to Proxmox and VMs.
- Internal bridges left without IP to enforce network segmentation.
- VM IDs, network mappings, and IP assignments documented for reference.
- Boot issue resolved by using VGA instead of serial console.
- Infrastructure prepared for pfSense, SOC components, and Active Directory VMs.
