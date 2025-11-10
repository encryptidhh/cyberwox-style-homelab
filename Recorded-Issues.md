# 1. Initial Setup
* Goal: Set up a CyberWOX-style homelab for cybersecurity practice, using VMware VMs.
* Main components:
  * VMWare (VMware Workstation 17.5.2)
  * pfSense/OPNsense Firewall
  * Security Onion (for monitoring and analysis)
  * Splunk Server
  * Windows Domain Controller and Clients
  * Kali Linux Machine
 
# 2. Firewall Configuration Issues
## pfSense Firewall
* Issue: Unable to extract the ISO file.
* Fix: Switched to OPNsense (DVD format) as an alternative.
## OPNsense installation
* Issue: Could not extract firewall image as a .dv2 file.
    * Tried using **bzip2** as suggested by a Medium article - [link here](https://medium.com/@akobeajiboluemmanuel/opnsense-firewall-setup-and-installation-on-vmware-10a3057b07aa).
    * Didn't work - produced blank lines/hanging output.
* Fix: Used **7-Zip** on my host machine to extract the ISO successfully.
## Installation Mode
* Issue: “Not enough disks to install.”
* Fix: Changed from ZFS (requires multiple disks) to UEFI mode (no disks required).
## Login Credentials
* Issue: Default credentials (`installer` / `opnsense`) not working.
* Fix: Used `root` as login and password as `password` (since I changed it during setup).
## GUI Connection via a Kali Machine
* Issue: Couldn’t connect to the OPNsense Firewall GUI.
* Fix:
  * Changed Kali Linux network settings → switched to VMNet1.
  * Set a static IP within **192.168.1.0/24** (or any subnet range).
  * Steps to change Kali's static config:
    1. Go to **Advanced Network Configurations** → **IPv4 Settings**.
    2. Set mode to Static.
    3. Use CIDR notation for subnet mask (/24 for 255.255.255.0).
    4. Default gateway = **OPNsense IP** (192.168.1.1).
    5. Verify IP via `hostname -I` or `ifconfig`.
    6. Test with ping [OPNsense IP].
    7. Open Firefox → type in OPNsense IP to access the GUI.
* Result: GUI connection successful, no timeout errors.
  
# 3. Security Onion and Management Machine
## SecOnionMgmt Placement
* Issue: Placed the SecOnionMgmt Ubuntu machine on the same subnet as Kali.
* Fix: Should’ve connected via **Bridged** network for Internet access.
  * Used command `sudo so-allow` on Security Onion to authorize connection.
## Missing Commands and Apps
* Issue: `so-allow`, `so-firewall`, and the SOC web database were unavailable.
* Fix: Uninstalled and reinstalled Security Onion.
* Ensured sufficient system resources for smooth installation:
  * Storage ≥ **200 GB**
  * Memory ≥ **8 GB**
  * CPU processors ≥ **4** (not cores).
## Connectivity Problems
* Issue: SecOnionMgmt couldn’t access Security Onion GUI even though ping worked both ways.
* Fix: Placed both VMs on the same internet-accessible subnet (VMNet8 - NAT).
* Added management IP to Security Onion firewall:
  
```
sudo so-firewall include analyst <SecOnionMgmt IP>
sudo so-firewall apply
```
* Verified GUI access worked afterward via Firefox on the Kali Machine.

# 4. Active Directory Setup
## AD Certificate Services
* Issue: “Red notification (1)” under AD CS Wizard.
* Fix: Enable Log on as a service in Group Policy:
```
Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Local Policies -> User Rights Assignment
```
## Domain Access
* Issue: Windows machines couldn’t access the domain.
* Fix (multi-step):
  1. From Kali → Access OPNsense GUI → go to em2VictimNetwork.
  2. Add the Domain Controller’s IP (check with `ipconfig` first).
  3. On the Domain Controller: confirm it’s set to Domain mode ([YOURname].local).
  4. Add [YOURname].local as the DNS Server Name in OPNsense.
  5. On Windows client → Control Panel → Network and Sharing Center → Ethernet0 Properties.
  6. Under TCP/IPv4 Properties, statically set:
     * IP address
     * Subnet mask
     * Preferred DNS Server = Domain Controller IP
  7. Disable Windows Firewall on both the DC and the client.
  8. Rejoin the machine to the domain.
  9. When prompted, use credentials like `[YOURname]\Username`.

 # 5. Ubuntu Server + Splunk Configuration
 ## Internet Connectivity
 * Issue: tasksel couldn’t run; system couldn’t install packages.
 * Cause: No IP assigned — network interfaces not configured.
 * Fix:
   1. Switched interface to VMNet8 (NAT) for Internet.
   2. Added VMNet6 to connect `SplunkServer` to OPNsense.
   3. Reloaded `SplunkServer` → checked IPs via `ip a`.
* Steps taken after reinstallation:
 1. Created a YAML config file:
```
sudo nano /etc/netplan/initial-config-file
```
  2.  Added the following configuration into the YAML file:   
```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: true
```
  3. And applied the relevant config:
```
sudo netplan apply
sudo systemctl restart systemd-networkd
```
* Verified VMware NAT service was running on my **host machine**
```
services.msc → VMware NAT Service → Restart/Start.
```
* Checked for updates within the `SplunkServer` Ubuntu machine:
```
sudo apt install tasksel
sudo tasksel ubuntu-desktop
```
* **If that fails**, run:
```
sudo apt install ubuntu-desktop
reboot
```
* Confirmed GUI installation by watching for GUI-only apps (e.g., Thunderbird).
# 6. Splunk Forwarder Communication
* Issue: Splunk Forwarder and Splunk Instance not communicating.
* Observation: Windows machine had an APIPA address, meaning a static IP config was needed.
* Fix: Navigate to Networking Settings within Windows Control Panel:
```
Control Panel -> Networking and Sharing Center -> Ethernet0
```
  * Within the Ethernet0 settings, set:
    * IP address = any IP within the range of 192.168.6.2-255
    * Subnet mask = 255.255.255.0
    * Default gateway = 192.168.6.1 (Splunk Instance IP address)
