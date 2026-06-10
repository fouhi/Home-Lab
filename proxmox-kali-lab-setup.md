# Proxmox Cybersecurity Lab — Kali Setup & Network

## Lab Architecture

```
Internet
    │
  vmbr0 (192.168.10.x) ←── Proxmox host
    │
  [Kali Attacker VM] ── also on ──► vmbr1 (192.168.100.x, isolated)
                                          │
                                     [Docker VM]  ← DVWA :8080, Juice Shop :3000
                                     [Metasploitable] (later)
```

- **vmbr0** — internet-facing bridge, Proxmox host lives here
- **vmbr1** — isolated lab bridge, NO gateway, NO internet, lab targets only

---

## Step 1 — Create Isolated Lab Bridge (vmbr1)

1. Proxmox web UI → **Node → System → Network → Create → Linux Bridge**
2. Fill in:

| Field         | Value              |
|---------------|--------------------|
| Name          | `vmbr1`            |
| IPv4/CIDR     | *(leave blank)*    |
| Gateway       | *(leave blank)*    |
| Autostart     | ✅ checked         |
| Bridge ports  | *(leave blank)*    |

3. Click **Create** → **Apply Configuration**
4. Verify in Proxmox Shell:
```bash
ip link show vmbr1
# Should show: UP
```

---

## Step 2 — Import Kali Linux (.qcow2)

### 2a — Transfer .qcow2 to Proxmox host
```bash
# From your local machine
scp kali-linux-*.qcow2 root@<proxmox-ip>:/var/lib/vz/images/
```

### 2b — Create empty VM (no disk)

**Create VM** wizard settings:

| Tab       | Setting                          |
|-----------|----------------------------------|
| General   | Name: `kali-attacker`            |
| OS        | Do not use any media             |
| System    | Enable **Qemu Agent** checkbox   |
| Disks     | **Delete the default disk**      |
| CPU       | Cores: `2`, Type: `host`         |
| Memory    | `8192` MB                        |
| Network   | Bridge: `vmbr0`, Model: VirtIO   |

Note your **VM ID** (e.g. `100`).

### 2c — Import the disk
```bash
# Run in Proxmox Shell
qm importdisk 100 /var/lib/vz/images/kali-linux-*.qcow2 local-lvm
```

### 2d — Attach imported disk
1. VM → **Hardware** → double-click **Unused Disk 0** → **Add**
2. VM → **Options** → **Boot Order** → enable the disk, move to top

---

## Step 3 — Add Second NIC (vmbr1)

1. VM → **Hardware** → **Add** → **Network Device**
2. Bridge: `vmbr1`, Model: **VirtIO**
3. Click **Add**

Kali now has:
- `eth0` / `ens18` → `vmbr0` (internet)
- `eth1` / `ens19` → `vmbr1` (lab targets)

---

## Step 4 — Performance Settings

In VM **Hardware**, verify/change:

| Setting       | Value                        |
|---------------|------------------------------|
| Network Model | `VirtIO (paravirtualized)`   |
| CPU Type      | `host`                       |
| RAM           | `8192` MB                    |
| Display       | `VirtIO-GPU` or `SPICE`      |

---

## Step 5 — First Boot & Setup

Default credentials: `kali` / `kali`

```bash
# Change password immediately
passwd

# Update everything
sudo apt update && sudo apt full-upgrade -y

# Install QEMU guest agent (Proxmox integration)
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent

# Run VM optimisation tool
sudo kali-tweaks
# → Virtualization → apply recommended settings
```

Back in Proxmox UI:
- VM → **Options** → **QEMU Guest Agent** → enable it

---

## Step 6 — Configure Lab Network Interface

Set a static IP on Kali's second NIC for the lab network:

```bash
# Bring up the lab interface (replace ens19 with your actual interface name)
sudo ip addr add 192.168.100.20/24 dev ens19
sudo ip link set ens19 up

# Verify
ip a show ens19
```

To make it persistent, edit `/etc/network/interfaces`:
```
auto ens19
iface ens19 inet static
    address 192.168.100.20
    netmask 255.255.255.0
```

---

## Step 7 — Snapshot

In Proxmox → right-click VM → **Snapshots** → **Take Snapshot**
- Name: `kali-baseline`
- Description: `Clean install, updated, guest agent installed`

> ⚠️ Always snapshot before each lab session so you can roll back after testing exploits.

---

## Planned IP Scheme

| VM                | Bridge  | IP Address       |
|-------------------|---------|------------------|
| Proxmox host      | vmbr0   | 192.168.10.2     |
| Kali (internet)   | vmbr0   | DHCP             |
| Kali (lab)        | vmbr1   | 192.168.100.20   |
| Docker VM (lab)   | vmbr1   | 192.168.100.10   |
| Metasploitable    | vmbr1   | 192.168.100.30   |

---

## Useful Kali Tools for the Lab

| Tool         | Use Case                                      |
|--------------|-----------------------------------------------|
| Burp Suite   | Intercept HTTP traffic → DVWA, Juice Shop     |
| sqlmap       | Automated SQL injection testing               |
| nikto        | Web server vulnerability scanning             |
| gobuster     | Directory & endpoint brute-forcing            |
| ffuf         | Fast web fuzzer                               |
| Metasploit   | Exploit framework (great with Metasploitable) |
| wpscan       | WordPress vulnerability scanner               |

---

## TODO — Next Steps

- [ ] Create Docker VM (Debian 12, vmbr1 only, static IP 192.168.100.10)
- [ ] Install Docker + Docker Compose inside Docker VM
- [ ] Deploy DVWA (`http://192.168.100.10:8080`)
- [ ] Deploy OWASP Juice Shop (`http://192.168.100.10:3000`)
- [ ] Import Metasploitable 2 (vmbr1 only)
- [ ] Add WebGoat container
- [ ] Import VulnHub machines for CTF practice
