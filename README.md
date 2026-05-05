# Building a Headless Ad-Blocking Fortress: Pi-hole + Tailscale on Raspberry Pi OS "Trixie"

Setting up a Raspberry Pi as a network-wide ad blocker and VPN server is a rite of passage. But if you're running the bleeding-edge Debian 13 ("Trixie") underneath Raspberry Pi OS Lite, or you are trapped behind your ISP's Carrier-Grade NAT (CGNAT), the classic installation scripts will fail.

Here is how I built my headless setup, bypassed the static IP installer crash, and tunneled through my ISP's CGNAT wall.

## The Goal

* **Pi-hole:** Block ads and trackers across the entire home Wi-Fi network.
* **Tailscale:** Create a zero-config VPN to securely tunnel back into my home network from my phone to get ad-blocking on the go (completely bypassing CGNAT).

## Step 1: The Headless Setup (Raspberry Pi Imager)
Because I didn't want to plug a monitor into the Pi, I used the Raspberry Pi Imager on my PC to bake the credentials right into the SD card.

* Selected Raspberry Pi OS Lite (no desktop environment = less RAM usage).
* Clicked Edit Settings (the gear icon) before writing to the SD card.
* Set the Hostname, created a Username/Password, and entered my Wi-Fi credentials.
* **Crucial:** Under the "Services" tab, I checked Enable SSH.

Once the SD card was in the Pi and powered on, it connected to my network automatically. I found its IP address via my router's admin page and SSH'd in:

    ssh my_username@192.168.1.50
Step 2: Securing the Perimeter (UFW)

Before installing the stuff, I locked down the server with Uncomplicated Firewall (UFW):

    sudo apt update && sudo apt upgrade -y
    sudo apt install ufw -y
    sudo ufw allow ssh          # Critical! Do not skip this.
    sudo ufw allow 80/tcp       # Pi-hole Web Admin
    sudo ufw allow 53/tcp       # DNS
    sudo ufw allow 53/udp       # DNS
    sudo ufw enable

Step 3: The "Trixie" Static IP Gotcha

Here is where things got interesting. I tried to run the standard Pi-hole install script (curl -sSL https://install.pi-hole.net | bash), but it crashed, complaining about my network environment.
The Problem: The newest Raspberry Pi OS (based on Debian 13 "Trixie") uses NetworkManager for networking. The Pi-hole installer script expects the older Debian network tools and panics when trying to assign a static IP.
The Fix: Manually set the static IP using nmtui before running the script.

    Run sudo nmtui in the terminal.
    Go to Edit a connection -> Select the active network (Wired or Wi-Fi).
    Scroll down to IPv4 CONFIGURATION and change <Automatic> to <Manual>.
    Expand <Show> and enter the static IP with the subnet mask (e.g., 192.168.1.237/24). Add the gateway (router IP) and a temporary DNS like 1.1.1.1.
    Save, quit, and force the Pi to apply it by rebooting: sudo reboot

Step 4: Installing Pi-hole
Once reconnected via SSH to the new static IP, the installer script sailed right through without any network panics.
Bash

    curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

I followed the blue prompts, chose Cloudflare as my upstream DNS, and made sure to save the Admin Webpage Password generated at the end

Step 5: Bypassing CGNAT with Tailscale

Because my ISP puts my network behind Carrier-Grade NAT (CGNAT), traditional port-forwarding for VPNs like WireGuard is impossible. Tailscale uses WireGuard under the hood but easily punches through this wall.
Run the official install script:

    curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh

Start Tailscale and authenticate the Pi via the link generated in the terminal:

    sudo tailscale up

Tell our UFW firewall to trust the new virtual Tailscale network:

    sudo ufw allow in on tailscale0

Grab the Pi's new Tailscale IP address (starts with 100.x.x.x) to use in the next step:

    tailscale ip -4

Step 6: Flipping the Switch (DNS Configuration)

To make Pi-hole and Tailscale talk to each other, both systems need to be configured.
Configure Pi-hole Interface:

    Open the Pi-hole web dashboard (http://192.168.1.237/admin).
    Navigate to Settings -> DNS tab.
    Under Interface settings, change it to Permit all origins (This is safe because UFW blocks unwanted outside traffic). Click Save.

Configure Tailscale Admin Console:

    Log into the Tailscale Web Console (tailscale.com).
    Go to the DNS tab.
    Under Nameservers, click Add Nameserver -> Custom...
    Paste the Pi's Tailscale IP address (100.x.x.x).
    Check the toggle for Override local DNS.

4. Configure the Home Router (Optional but Recommended):

To get ad-blocking on smart TVs and laptops at home without needing the Tailscale app, log into your home router's DHCP settings and change the Primary DNS to the Pi's local IP address (192.168.1.237). Remove any Secondary DNS to ensure devices don't bypass the Pi-hole.
After connecting my phone to Tailscale on cellular data, the dashboard graphs lit up. Success!
