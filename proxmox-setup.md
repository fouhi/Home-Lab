# Proxmox VE 9 — Post-Install Setup Guide

> Tested on Proxmox VE 9.x (Debian Trixie). For home lab / cybersecurity practice use.

---

## 1. Access the Web UI

Go to `https://YOUR_IP:8006` in your browser. Accept the self-signed certificate warning.

- **Username:** `root`
- **Realm:** `Linux PAM`
- **Password:** set during installation

Dismiss the "No valid subscription" popup.

---

## 2. Fix Repositories

By default Proxmox points to the paid enterprise repo — this needs to be switched to the free no-subscription repo.

### 2a. Disable the enterprise repos

Open each file and comment out every line (add `#` at the start):

```bash
nano /etc/apt/sources.list.d/pve-enterprise.sources
nano /etc/apt/sources.list.d/ceph.sources
```

### 2b. Create the no-subscription repo

> ⚠️ On Proxmox 9 / Trixie, use the `.sources` format — **not** `.list`

```bash
nano /etc/apt/sources.list.d/pve-no-subscription.sources
```

Paste this inside:

```
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

> The warning *"not recommended for production use"* is normal — perfectly fine for home lab use.

---

## 3. Update the System

```bash
apt update && apt upgrade -y
reboot now
```

---

## 4. Remove the Subscription Nag Popup

```bash
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && \
  systemctl restart pveproxy.service
```

Clear browser cache and log back in.

---

## 5. Configure DNS

```bash
nano /etc/resolv.conf
```

Make sure a nameserver is present, e.g.:

```
nameserver 1.1.1.1
```

---

## 6. Set Up Storage

Default storage:
- `local` — ISOs and backups
- `local-lvm` — VM disk images

To add more storage: **Node → Disks → LVM → Create: LVM Volume Group**

For network storage (NAS): **Datacenter → Storage → Add → NFS / SMB**

---

## 7. Enable VLAN Awareness (optional but useful)

Allows per-VM VLAN tagging on a single bridge.

**Node → System → Network → vmbr0 → Edit → check "VLAN aware" → Apply**

---

## 8. Enable PCI Passthrough (optional)

Only needed if passing a GPU or PCIe device to a VM.

```bash
nano /etc/default/grub
```

Edit the `GRUB_CMDLINE_LINUX_DEFAULT` line:

```
# Intel:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# AMD:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

Then:

```bash
update-grub
nano /etc/modules
```

Add:

```
vfio
vfio_iommu_type1
vfio_pci
```

Reboot.

---

## 9. Basic Security — Fail2Ban

```bash
apt install fail2ban -y
systemctl enable --now fail2ban
```

Edit `/etc/fail2ban/jail.conf` to set `bantime` and `maxretry`, then:

```bash
systemctl restart fail2ban
```

---

## 10. Create Your First VM

1. Upload ISO: **local → ISO Images → Upload**
2. Click **Create VM** (top right) and follow the wizard
3. After OS install, install the QEMU Guest Agent inside the VM:

**Linux:**
```bash
apt install qemu-guest-agent && systemctl enable --now qemu-guest-agent
```

**Windows:** Install the VirtIO drivers ISO from the Proxmox wiki.

---

## 11. Create Your First LXC Container

1. Download a template: **local → CT Templates → Templates**
2. Click **Create CT** and follow the wizard
3. Unprivileged containers are recommended for security

---

## Quick Reference

| Task | Location |
|------|----------|
| Manage repos | Node → Updates → Repositories |
| Add storage | Datacenter → Storage → Add |
| Network config | Node → System → Network |
| Firewall | Datacenter / Node → Firewall |
| Backup config | Datacenter → Backup |
| Task logs | Node → Task History |

---

*Proxmox VE docs: https://pve.proxmox.com/wiki/Main_Page*
