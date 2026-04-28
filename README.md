Building a Headless Ad-Blocking Fortress: Pi-hole + WireGuard on Raspberry Pi OS "Trixie"
Setting up a Raspberry Pi as a network-wide ad blocker and VPN server is a rite of passage. But if you're running the bleeding-edge Debian 13 ("Trixie") underneath Raspberry Pi OS Lite, the classic installation scripts might throw some curveballs.
Here is how I built my headless setup , and how I bypassed the static IP installer crash.
The Goal:

    Pi-hole: Block ads and trackers across the entire home Wi-Fi network.
    WireGuard : Securely tunnel back into my home network from my phone to get ad-blocking on the go.

Step 1: The Headless Setup (Raspberry Pi Imager)
Because I didn't want to plug a monitor into the Pi, I used the Raspberry Pi Imager on my PC to bake the credentials right into the SD card.

    Selected Raspberry Pi OS Lite (no desktop environment = less RAM usage).
    Clicked Edit Settings (the gear icon) before writing to the SD card.
    Set the Hostname, created a Username/Password, and entered my Wi-Fi credentials.
    Crucial: Under the "Services" tab, I checked Enable SSH.

Once the SD card was in the Pi and powered on, it connected to my network automatically. I found its IP address via my router's admin page and SSH'd in:

    ssh my_username@192.168.1.50

Step 2: Securing the Perimeter (UFW)
Before installing the stuff, I locked down the server with Uncomplicated Firewall:

    sudo apt update && sudo apt upgrade -y
    sudo apt install ufw -y
    sudo ufw allow ssh          Critical!
    sudo ufw allow 80/tcp       Pi-hole Web Admin
    sudo ufw allow 53/tcp       DNS
    sudo ufw allow 53/udp       DNS
    sudo ufw allow 51820/udp    WireGuard VPN
    sudo ufw enable

Step 3: The "Trixie" Static IP Gotcha
Here is where things got interesting. I tried to run the standard Pi-hole install script (curl -sSL https://install.pi-hole.net | bash), but it crashed, complaining about my network environment.
The Problem: The newest Raspberry Pi OS (based on Debian 13 "Trixie") uses NetworkManager for networking. The Pi-hole installer script expects the older Debian network tools and panics when trying to assign a static IP.

The Fix: Manually set the static IP using nmtui before running the script:

    Run sudo nmtui in the terminal.
    Go to Edit a connection -> Select the active network (Wired or Wi-Fi).
    Scroll down to IPv4 CONFIGURATION and change <Automatic> to <Manual>.
    Expand <Show> and enter the static IP with the subnet mask (e.g., 192.168.1.237/24). Add the gateway (router IP) and a temporary DNS like 1.1.1.1.
    Save, quit, and force the Pi to apply it by rebooting: sudo reboot

Step 4: Installing Pi-hole
Once reconnected via SSH to the new static IP, the installer script sailed right through without any network panics.

    curl -sSL https://install.pi-hole.net | bash

I followed the blue prompts, chose Cloudflare as my upstream DNS, and made sure to save the Admin Webpage Password generated at the end.
Step 5: Setting up WireGuard 

To get these benefits outside my house, I installed WireGuard. It's a wrapper that configures WireGuard effortlessly.

    curl -L https://install.pivpn.io | bash

During the setup:

    I selected WireGuard as the protocol.
    The script auto-detected Pi-hole and asked if I wanted to use it as the DNS server for the VPN. I selected Yes.

After a reboot, I generated a client profile for my phone:

    pivpn add       # Name the client
    pivpn -qr       # Generate a QR code in the terminal

I scanned the  QR code with the WireGuard app on my phone, and the tunnel was instantly configured. (Note: You must port-forward UDP 51820 on your router to the Pi for the VPN to work from the outside world).
Step 6: Flipping the Switch (Router Config)

The final step was telling my home router to send all network traffic to the Pi-hole:

    Logged into my router's admin page.
    Found the DHCP/LAN Settings.
    Changed the Primary DNS to the Pi-hole's IP address (192.168.1.237).
    Erased the Secondary DNS entirely (so smart devices can't bypass the Pi-hole).
    Added a DHCP Reservation binding the Pi's MAC address to its IP, ensuring the router never hands that IP to another device.
After restarting the router to flush the network, the dashboard graphs lit up.
<img width="1918" height="976" alt="Screenshot 2026-04-28 162813" src="https://github.com/user-attachments/assets/8356d554-a38e-4fa8-b9dd-67d5d66c6a18" />


