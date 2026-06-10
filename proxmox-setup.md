# Proxmox VE 9 — Post-Install Setup Guide

> Tested on Proxmox VE 9.x (Debian Trixie). For home lab  use.

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


---

## 7. Enable VLAN Awareness (optional but useful)

Allows per-VM VLAN tagging on a single bridge.

**Node → System → Network → vmbr0 → Edit → check "VLAN aware" → Apply**


---

## 8. Basic Security — Fail2Ban

```bash
apt install fail2ban -y
systemctl enable --now fail2ban
```

Edit `/etc/fail2ban/jail.conf` to set `bantime` and `maxretry`, then:

```bash
systemctl restart fail2ban
```

---

*Proxmox VE docs: https://pve.proxmox.com/wiki/Main_Page*
