# Building a Headless Ad-Blocking Fortress: Pi-hole + Tailscale on Raspberry Pi OS Trixie

This guide documents a headless Raspberry Pi setup that runs Pi-hole for network-wide ad blocking and Tailscale for secure remote access without port forwarding. It focuses on Raspberry Pi OS based on Debian 13 "Trixie", where the Pi-hole installer can fail if the network is not prepared first.

## Goal

- Use **Pi-hole** as the main DNS server for ad and tracker blocking on the home network.
- Use **Tailscale** to reach the home network remotely and keep Pi-hole filtering active outside the house.
- Avoid problems caused by CGNAT, which breaks traditional inbound VPN setups.

## Hardware and software

- Raspberry Pi running Raspberry Pi OS Lite.
- Pi-hole.
- Tailscale.
- SSH enabled for headless access.
- A router or VLAN-capable network handing out the Pi-hole IP as DNS.

## Step 1: Prepare the Pi headlessly

Using Raspberry Pi Imager makes it possible to bring the Pi online without a monitor or keyboard.

1. Open Raspberry Pi Imager.
2. Select Raspberry Pi OS Lite.
3. Open the advanced settings before writing the SD card.
4. Set the hostname.
5. Create a username and password.
6. Add Wi-Fi credentials if you are not using Ethernet.
7. Enable SSH.
8. Write the image to the SD card.

After first boot, find the Pi's IP address from your router or network controller and connect over SSH:

```bash
ssh my_username@192.168.1.50
```

## Step 2: Update the system and enable a basic firewall

Before installing services, update the system and open only the ports you actually need.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw -y
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw enable
```

Port 80 is needed for the Pi-hole web interface. Port 53 TCP and UDP are needed for DNS.

## Step 3: Fix the static IP problem on Trixie

On Raspberry Pi OS Trixie, the Pi-hole installer may fail if the network is still using automatic addressing. The clean fix is to assign the static address first, then run the installer.

If the system uses NetworkManager, use `nmtui`:

```bash
sudo nmtui
```

Then:

1. Choose **Edit a connection**.
2. Select the active connection, usually `eth0` for Ethernet.
3. Change IPv4 configuration from automatic to manual.
4. Enter a static IP, for example `192.168.1.237/24`.
5. Set the gateway to your router IP, for example `192.168.1.1`.
6. Set a temporary DNS server such as `1.1.1.1`.
7. Save and exit.
8. Reboot the Pi.

```bash
sudo reboot
```

After reboot, reconnect with SSH using the new static IP.

## Step 4: Install Pi-hole

Once the network is stable, install Pi-hole with the official script.

```bash
curl -sSL https://install.pi-hole.net | bash
```

During installation:

- Choose an upstream DNS provider such as Cloudflare.
- Enable the web admin interface.
- Save the generated admin password.

After installation, open the web interface:

```text
http://192.168.1.237/admin
```

Replace the IP with your real Pi-hole address if you changed it later.

## Step 5: Install Tailscale to bypass CGNAT

If your ISP uses CGNAT, classic port forwarding for WireGuard or similar VPNs may not work. Tailscale avoids that problem by creating an encrypted mesh connection between your devices.

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start it:

```bash
sudo tailscale up
```

Authorize the node in the browser using the login link shown in the terminal.

Then allow the Tailscale interface through UFW:

```bash
sudo ufw allow in on tailscale0
```

Get the Pi's Tailscale IPv4 address:

```bash
tailscale ip -4
```

This usually returns an address in the `100.x.x.x` range.

## Step 6: Make Pi-hole work for both local and remote clients

For Pi-hole and Tailscale to work together, both sides need to be configured correctly.

### Pi-hole settings

1. Open the Pi-hole dashboard.
2. Go to **Settings** -> **DNS**.
3. Under interface listening behavior, set it to **Permit all origins**.
4. Save the changes.

This allows the Pi-hole service to answer DNS requests coming from outside its local subnet, such as from Tailscale clients or other VLANs.

### Tailscale DNS settings

1. Open the Tailscale admin console.
2. Go to **DNS**.
3. Add a custom nameserver.
4. Enter the Pi's Tailscale IP address.
5. Enable **Override local DNS** if you want connected Tailscale devices to use Pi-hole automatically.

## Step 7: Configure your local network to use Pi-hole

To make devices at home use Pi-hole without installing the Tailscale app on every device, set the router or DHCP server to hand out the Pi-hole IP as the DNS server.

Recommended approach:

- Set the primary DNS server to the Pi-hole local IP.
- Remove the secondary DNS server unless you intentionally want a fallback.

If you leave a public DNS server as secondary, some devices will bypass Pi-hole completely.

## Example final layout

### Simple flat LAN

- Pi-hole IP: `192.168.1.237`
- Router IP: `192.168.1.1`
- Pi-hole web UI: `http://192.168.1.237/admin`

### VLAN-based network

If the Pi-hole host later moves into a dedicated VLAN, the same logic applies. For example:

- Pi-hole VLAN: `192.168.30.0/24`
- Pi-hole IP: `192.168.30.2`
- Gateway: `192.168.30.1`
- Router DHCP for all client VLANs should hand out `192.168.30.2` as DNS.

## Troubleshooting

### Pi-hole installer crashes on Trixie

Set the static IP first with `nmtui`, reboot, and then rerun the installer.

### DNS works locally but not over Tailscale

Set Pi-hole to **Permit all origins** and confirm that Tailscale DNS points to the Pi's Tailscale IP.

### Devices ignore Pi-hole

Remove any secondary DNS server from DHCP or router settings.

### SSH stops working after IP change

Reconnect to the new IP address instead of the old one.

## Result

With this setup, the Raspberry Pi acts as both a local DNS sinkhole and a remotely reachable DNS server through Tailscale. That gives ad blocking at home, remote filtered DNS when away, and no dependency on public port forwarding.
