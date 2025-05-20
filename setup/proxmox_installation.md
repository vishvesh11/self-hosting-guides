---

# Proxmox VE 8.x Installation Guide

This guide details the process of installing Proxmox Virtual Environment (PVE) on a dedicated server. Proxmox VE is a powerful open-source virtualization platform that allows you to run virtual machines (VMs) and Linux containers (LXCs) on a single host.

## Prerequisites

Before you begin, ensure you have the following:

- **Dedicated Server:** A physical computer to serve as your Proxmox host.
  - **Processor:** 64-bit CPU (proxmox may not work on arm).
  - **RAM:** Minimum 2GB, but 8GB+ recommended for running multiple VMs/LXCs.
  - **Storage:** At least one drive for the Proxmox OS installation.
  - **Ethernet Ports:**
    - **Minimum 1 Gigabit Ethernet Port:** Required for Proxmox management access and network connectivity for your VMs/LXCs.
    - **2 Gigabit Ethernet Ports (Recommended):** Ideal if you plan to run pfSense to separate your WAN and LAN connections. My personal setup uses 3 ethernet ports.
- **USB Drive:** A USB flash drive (8GB or larger recommended) to create a bootable installer. **All data on this USB drive will be wiped.**
- **Another Computer (Laptop/Desktop):** A machine with internet access to download the Proxmox ISO and create the bootable USB drive. This guide assumes a Linux machine.
- **Command Line Interface (CLI) Experience:** Basic familiarity with Linux terminal commands.
- **Basic Understanding of Virtualization:** Knowledge of what Virtual Machines (VMs) and Linux Containers (LXCs) are.
- **Network Cable or Wi-Fi Ap:** To connect your Proxmox server to your router/switch.

## Installation Steps

### Part 1: Preparing the Bootable USB Drive

1.  **Download Proxmox VE ISO:**
    Visit the official Proxmox website:
    `https://www.proxmox.com/en/downloads/category/iso-images-pve`
    Download the latest stable Proxmox VE 8.x ISO installer (e.g., `proxmox-ve_8.1-2.iso`).

2.  **Identify Your USB Drive:**
    Connect your USB drive to your Linux machine. Open a terminal and run the following command to list all block devices:

    ```bash
    lsblk
    ```

    should look something like this

    ````
    root@vishu:~# lsblk
        NAME                         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
        sdb                            8:16   0 31.5G  0 disk
        └─sdb1                         8:17   0 31.5G  0 part ```
    Carefully examine the output to identify your USB drive. Look for its size and mount points (it might not have any). It will typically appear as `/dev/sdX` (e.g., `/dev/sdb`, `/dev/sdc`). **It is crucial to correctly identify your USB drive; writing to the wrong device will wipe your hard drive\!**
    For example, if your USB drive is 32GB and shows up as `sdb`, then its path is `/dev/sdb`.

    ````

3.  **Format the USB Drive (Optional but Recommended):**
    While `dd` (used in step 5) will overwrite the entire drive, it's good practice to ensure it's unmounted and clean. If your USB drive is mounted, unmount it first. Replace `/dev/sdX` with your identified USB drive.

    ```bash
    sudo umount /dev/sdX* # Unmount all partitions on the USB drive
    sudo wipefs -a /dev/sdX # Remove all partition signatures (clean slate)
    ```

4.  **Make the Bootable Drive (Using `dd`):**
    This command will write the Proxmox ISO image directly to your USB drive, making it bootable. **Double-check `/dev/sdX` is correct before executing this command\!**

    ```bash
    sudo dd if=proxmox-ve_8.1-2.iso of=/dev/sdX bs=1M status=progress conv=fsync
    ```

    - Replace `proxmox-ve_8.1-2.iso` with the actual name of your downloaded ISO file.
    - Replace `/dev/sdX` with the correct path to your USB drive (e.g., `/dev/sdb`).
    - `bs=1M` sets the block size to 1MB for faster transfer.
    - `status=progress` shows the progress of the write operation (useful for large files).
    - `conv=fsync` ensures all buffered data is written to the disk before the command exits.

    This process will take a few minutes depending on your USB drive's speed.

    **For Windows Users:**
    If you are on a Windows machine, you can use a tool like [Rufus](https://rufus.ie/en/) or [Balena Etcher](https://www.balena.io/etcher/) to create the bootable USB drive from the ISO. These tools provide a graphical interface for a more straightforward process.

### Part 2: Installing Proxmox VE on the Server

1.  **Prepare the Server:**
    Eject the USB drive safely from your PC. Plug the newly created bootable USB drive into your server. Ensure that **any data on your server's intended boot drive(s) that you wish to keep has been backed up**, as the installation process will wipe them.

2.  **Boot from USB:**
    Power on your server. Immediately start pressing the designated key to enter the **Boot Menu** (commonly `F10`, `F11`, `F12`) or **BIOS/UEFI Setup** (commonly `F2`, `Del`, `Esc`). In the Boot Menu, select your USB drive as the boot device. If you're in BIOS/UEFI, navigate to the Boot Order section and set the USB drive as the primary boot device. Save and exit.

3.  **Proxmox Installer Welcome:**
    The server will boot from the USB drive. You should see the Proxmox VE boot menu. Select "Install Proxmox VE" and press Enter.

4.  **Accept EULA:**
    Read and accept the End User License Agreement (EULA) by clicking "I Agree".

5.  **Select Target Hard Disk & Filesystem:**

    - **Target Harddisk:** Choose the drive(s) where Proxmox VE will be installed. **This will wipe all data on the selected drive(s).**
    - **Filesystem:**
      - **ZFS (Recommended for Redundancy and Advanced Features):** Select `ZFS (RAIDx)` if you have multiple drives and want data integrity, snapshots, and software RAID. You'll then choose your RAID level (e.g., `RAID1` for mirroring with two drives, `RAID10` for striping and mirroring with four+ drives). This is my personal preference for robustness.
      - **ext4 or XFS:** If you are installing on a single drive or prefer hardware RAID, `ext4` or `XFS` are suitable choices. `XFS` is often preferred for large files and heavy I/O workloads.

6.  **Country, Time Zone, and Keyboard Layout:**
    Select your appropriate Country, Time Zone, and Keyboard Layout. This will configure the system's locale settings.

7.  **Set Root Password and Email:**
    Choose a strong password for the `root` user. This password will be used to log in to the Proxmox web interface and the server's CLI.
    Provide a valid email address. This is used for system notifications and alerts. **Make sure you save this password in your password manager (like Vaultwarden\!).**

8.  **Network Configuration:**
    This is a crucial step.

    - **Management Interface:** Select the Ethernet port you want to use for Proxmox's web management interface (e.g., `enp0s31f6`, `eno1`).
    - **Hostname:** Enter a unique hostname for your Proxmox server (e.g., `pve-host`).
    - **IP Address:** Set a static IP address for your Proxmox server. **This is highly recommended for stable management access.**
      - **Example:** `192.168.5.2` (as in my case).
      - **Important:** Choose an IP address within your router's subnet that is **outside its DHCP range** to avoid conflicts. You can typically find your router's IP address (e.g., `192.168.1.1` for Airtel, `192.168.29.1` for Jio) by checking your currently connected device's network settings. Log in to your router's administration panel to view its DHCP range and potentially assign a static IP reservation for your Proxmox server's MAC address.
    - **Netmask:** Usually `255.255.255.0` (or `/24`).
    - **Gateway:** Your router's IP address (e.g., `192.168.5.1`, `192.168.1.1`, or `192.168.29.1`).
    - **DNS Server:** Your router's IP, or public DNS servers like `1.1.1.1` (Cloudflare) or `8.8.8.8` (Google). Using public DNS can sometimes offer better resolution stability.

9.  **Reboot:**
    Once the installation is complete, remove the USB installation media when prompted and click "Reboot".

## Post-Installation

After rebooting, your Proxmox server will be ready\!

1.  **Access the Web Interface:**
    From another computer on the same network, open a web browser and navigate to:
    `https://YOUR_PROXMOX_IP:8006`
    (e.g., `https://192.168.5.2:8006`)

2.  **Login:**
    Log in using the `root` username and the password you set during installation. You might see a certificate warning; you can safely proceed as it's a self-signed certificate for your local network.

3.  **(Optional) Update Repositories:**
    By default, Proxmox uses an enterprise repository which requires a subscription. To get updates without a subscription, you'll need to disable the enterprise repository and enable the no-subscription repository via the CLI.
    Access your Proxmox server via SSH (`ssh root@YOUR_PROXMOX_IP`) or directly from the console, then:

    - **Disable Enterprise Repo:**
      ```bash
      echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" | sudo tee /etc/apt/sources.list.d/pve-no-subscription.list
      ```
    - **Enable No-Subscription Repo (if not already enabled):**
      ```bash
      sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/pve-enterprise.list
      ```
    - **Update System:**
      ```bash
      sudo apt update && sudo apt dist-upgrade -y
      ```

---

Good luck\!
