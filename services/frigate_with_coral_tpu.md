---

# Frigate NVR with Google Coral TPU on Proxmox LXC

This guide details the setup of Frigate, an open-source NVR with AI object detection, on a Proxmox Linux Container (LXC). This setup leverages a Google Coral Edge TPU for accelerated object detection, significantly improving performance.

**Disclaimer on Frigate Version:**
This guide specifically uses an older version of Frigate (`0.13.2`) which some users prefer for its UI. If you wish to install the _latest_ version of Frigate, the Docker Compose configuration will be different (refer to the [official Frigate documentation](https://www.google.com/search?q=https://docs.frigate.video/frigate/installation/%23docker-compose)).
## Prerequisites

- **Proxmox VE Installed:** Your Proxmox host is fully operational.
- **Google Coral Edge TPU:**
  - **USB Accelerator:** The most common and easiest to pass through.
  - **PCIe Accelerator:** Requires a PCIe slot on your Proxmox server and often more complex passthrough.
- **Network Cameras (RTSP/RTMP):** Ensure your cameras support RTSP/RTMP streams as Frigate requires these.
- **Sufficient Storage:** Plan for ample storage for video recordings (e.g., a ZFS pool or mounted NFS share available to your Proxmox host).
- **Basic CLI Experience:** Familiarity with Linux command-line operations.
- **Understanding of LXCs:** How they differ from VMs and their resource allocation.

## Part 1: Setting up the Privileged LXC with Docker

Frigate is recommended to be installed on bare metal due to its direct interaction with hardware for video decoding and object detection. However, running it in a **privileged LXC** on Proxmox is a viable and efficient alternative, offering good performance with less overhead than a full VM.

1.  **Create a Privileged LXC:**
    When creating your LXC in Proxmox, ensure you tick the **"Privileged container"** option. This is crucial for hardware passthrough (like the Coral TPU).

    - **Resource Allocation:**
      - **CPU Cores:** Allocate at least `2 CPU cores`, but `4 CPU cores` are often recommended for smoother video processing and AI detection.
      - **RAM:** Allocate at least `8 GB RAM`. Frigate can be memory-intensive, especially with multiple camera streams and longer retention.
      - **Disk Size:** Ensure ample disk space for your recordings within the LXC or plan to mount external storage.

2.  **Use Community Script for Docker Installation (Recommended):**
    The Proxmox community provides an excellent script to simplify Docker installation within an LXC. This guide uses an Alpine-based Docker LXC script, known for its small footprint.

    - **Access Proxmox Host Shell:** Open a shell to your Proxmox VE host (e.g., via the Proxmox web interface `>_ Shell` icon for your `pve` node, or SSH into `root@YOUR_PROXMOX_IP`).
    - **Download and Run Alpine Docker LXC Helper Script:**
      ```bash
      bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/alpine-docker.sh)"
      ```
    - **Follow Script Prompts:**
      - The script will guide you through the LXC creation process.
      - When it asks for "Advanced Config", choose **`y`**.
      - The first advanced option will be "Privileged Container". Use the **`spacebar`** to select this option (it should show `[X]`), then press `Enter` to proceed.
      - Complete the rest of the prompts (e.g., assigning IP, memory, disk size).

3.  **Update and Upgrade LXC Packages:**
    After the LXC is created and you've logged into it (e.g., via `ssh` or Proxmox console), ensure its package lists are up-to-date.

    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

4.  **Create Frigate Directory Structure:**
    Navigate to your home directory within the LXC and create a dedicated directory for your Frigate application files and configuration.

    ```bash
    cd ~
    mkdir -p ~/Downloads/programs/frigate
    cd ~/Downloads/programs/frigate
    ```

5.  **Clone Frigate GitHub Repository (for `docker-compose.yml`):**
    While you'll be using a specific Docker image version, cloning the repository is a convenient way to get a starting `docker-compose.yml` file and other potential reference files.

    ```bash
    git clone https://github.com/blakeblackshear/frigate.git . # The '.' clones into the current directory
    ```

6.  **Create/Edit `docker-compose.yml` for Frigate:**
    Now, create or modify the `docker-compose.yml` file in your current directory (`~/Downloads/programs/frigate`). Use a text editor like `vi` or `nano`.

    ```bash
    vi docker-compose.yml
    ```

    Press `i` to enter insert mode in `vi`, then paste the following configuration. Replace `your_password` with a strong password for Frigate's RTSP authentication.

    ```yaml
    version: "3.9"
    services:
      frigate:
        container_name: frigate
        # Privileged: This is often needed for full hardware access inside containers,
        # especially for direct device passthrough. Be cautious with privileged containers.
        privileged: true
        restart: unless-stopped
        image: ghcr.io/blakeblackshear/frigate:0.13.2 # Specific version for desired UI
        shm_size: "64mb" # Shared memory - adjust based on camera resolution and FPS
        devices:
          # USB Coral TPU passthrough. Add one line per USB Coral device.
          - /dev/bus/usb:/dev/bus/usb
          # PCIe Coral TPU passthrough. Add one line per PCIe Coral device (e.g., /dev/apex_0, /dev/apex_1).
          - /dev/apex_0:/dev/apex_0
          # Example for other hardware (uncomment and modify as needed):
          # - /dev/video11:/dev/video11 # For Raspberry Pi 4B Camera
          # - /dev/dri/renderD128:/dev/dri/renderD128 # Intel iGPU hwaccel, update for your hardware

        volumes:
          - /etc/localtime:/etc/localtime:ro # Syncs container time with host time
          - ./config:/config # Frigate configuration files
          - ./storage:/media/frigate # Recorded video storage
          - ./database:/data/db # Frigate database (sqlite)
          - type: tmpfs # Optional: Reduces SSD wear by using RAM for cache
            target: /tmp/cache
            tmpfs:
              size: 1000000000 # 1GB (adjust as needed for cache)

        ports:
          - "8971:8971" # Frigate Web UI
          - "5000:5000" # Internal unauthenticated API access. **Be careful with external exposure.**
          - "8554:8554" # RTSP feeds from Frigate
          - "8555:8555/tcp" # WebRTC over TCP
          - "8555:8555/udp" # WebRTC over UDP
        environment:
          # IMPORTANT: Change this password for RTSP authentication
          FRIGATE_RTSP_PASSWORD: "your_strong_password"
    ```

    - **Save and Exit `vi`:** Press `Esc`, then type `:wq` and press `Enter`.

## Part 2: Installing Google Coral TPU Drivers on Proxmox Node

For the LXC to access your Google Coral TPU, the necessary drivers must be installed on the **Proxmox host machine**, not within the LXC.

1.  **Access Proxmox Host Shell:**
    Connect to your Proxmox VE host via SSH (`ssh root@YOUR_PROXMOX_IP`) or use the web interface shell.

2.  **Follow Official Google Coral Documentation for Runtime:**
    It's crucial to install the correct runtime for your specific Coral device (USB or PCIe) on your Proxmox host. Refer to the official Google Coral documentation for the most up-to-date instructions.

    - **Google Coral Get Started Guide:** `https://coral.ai/docs/accelerator/get-started/#runtime-on-linux`
    - **Key steps from the guide (as of current date):**
      - **Add Coral APT repository:**
        ```bash
        echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
        curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        sudo apt update
        ```
      - **Install Edge TPU Runtime:**
        - For **USB Accelerator**:
          ```bash
          sudo apt install libedgetpu1-std
          ```
        - For **PCIe Accelerator**:
          ```bash
          sudo apt install libedgetpu1-max
          ```
        - _(Note: The `gasket-dkms` package might be automatically installed as a dependency. If not, or if you run into issues, see the troubleshooting section below.)_

3.  **Reboot the Proxmox Node:**
    After installing the drivers on the Proxmox host, it's essential to reboot the entire node to ensure the kernel modules are loaded correctly.

    ```bash
    sudo reboot
    ```

4.  **Verify Coral TPU Detection on Proxmox Host:**
    After the Proxmox node reboots, log back into its shell and use `lspci` to verify if the Coral TPU is detected.

    ```bash
    lspci
    ```

    - **Expected Output (for PCIe Coral):** You should see an entry similar to this, indicating the Coral Edge TPU:
      ```
      04:00.0 System peripheral: Global Unichip Corp. Coral Edge TPU
      ```
    - **For USB Coral:** You might not see it with `lspci`, but `lsusb` should show it:
      ```bash
      lsusb
      ```
      Look for "Google, Inc. GlobalFoundries Edge TPU".

### Troubleshooting: `gasket-dkms` and PCI Utilities

If `lspci` doesn't show your PCIe Coral, or you encounter issues, here are some common troubleshooting steps:

1.  **Install PCI Utilities:**
    Ensure `pciutils` is installed on your Proxmox host.

    ```bash
    sudo apt install pciutils
    ```

2.  **Install `lshw` (List Hardware):**
    A useful utility to list detailed hardware information.

    ```bash
    sudo apt install lshw
    ```

    Then run `sudo lshw -C system` or `sudo lshw -C display` to check hardware.

3.  **Switching Proxmox Repositories (if you haven't already):**
    If your Proxmox subscription is expired or you don't have one, you'll need to switch to the no-subscription repositories to get updates for system packages, including those needed for drivers. This should ideally be done _before_ installing drivers.

    - **Disable Enterprise Repository:**
      Edit the `pve-enterprise.list` file:

      ```bash
      sudo nano /etc/apt/sources.list.d/pve-enterprise.list
      ```

      Change the line from:
      `deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise`
      To:
      `# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise` (Comment out the line)
      **AND add:**
      `deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription`

    - **Disable Ceph Enterprise Repository (if applicable):**
      If you are using Ceph storage and have its enterprise repository enabled, you might need to switch that too.
      Edit the `ceph.list` file:

      ```bash
      sudo nano /etc/apt/sources.list.d/ceph.list
      ```

      **If it’s Ceph Quincy:**
      Change: `deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise`
      To: `# deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise`
      **AND add:**
      `deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription`

      **If it’s Ceph Reef:**
      Change: `deb https://enterprise.proxmox.com/debian/ceph-reef bookworm enterprise`
      To: `# deb https://enterprise.proxox.com/debian/ceph-reef bookworm enterprise`
      **AND add:**
      `deb http://download.proxmox.com/debian/ceph-reef bookworm no-subscription`

    - **Update Proxmox Host Packages:**

      ```bash
      sudo apt update && sudo apt upgrade -y
      ```

4.  **Fixing `gasket-dkms` Issues (Manual Install):**
    Sometimes, the `gasket-dkms` driver (necessary for PCIe Coral) doesn't install correctly or has issues. This manual compilation and installation can resolve it.

    ```bash
    # Remove any problematic existing gasket-dkms installation
    sudo apt remove gasket-dkms

    # Install build tools
    sudo apt install git devscripts dh-dkms -y

    # Clone the gasket driver source
    git clone https://github.com/google/gasket-driver.git
    cd gasket-driver/

    # Build the Debian package
    debuild -us -uc -tc -b

    # Navigate up to find the .deb package
    cd ..

    # Install the manually built package
    sudo dpkg -i gasket-dkms_1.0-18_all.deb # **Adjust version if it's different**

    # Update and upgrade again
    sudo apt update && sudo apt upgrade -y
    ```

    After these steps, **reboot your Proxmox node again** and check `lspci` to see if the Coral TPU is now detected.

Once your Coral TPU is correctly recognized by the Proxmox host, the LXC (because it's privileged and has the `/dev/bus/usb` or `/dev/apex_0` devices passed through) will be able to access it.
This is an excellent, detailed guide for configuring cameras in Frigate, especially with the practical advice on finding RTSP URLs. I'll refine it by adding more context, clear headings, and ensuring all commands and concepts are fully explained for someone following your guide.


## Frigate NVR: Camera Configuration Guide

This section will guide you through the process of connecting your IP cameras to your network, finding their RTSP streams, and configuring them within Frigate.

### Part 1: Discovering and Securing Your IP Cameras

Before adding cameras to Frigate, you need to ensure they are accessible on your local network and that you have their RTSP stream URLs.

1.  **Connect Cameras to Your Network & Assign Static IPs:**
    Connect all your IP cameras to your network router/switch. If your cameras were previously configured with an NVR, they might already have an assigned IP address.

    * **Find Camera IPs:**
        * Log in to your **router's web interface**.
        * Navigate to the **"Client List," "Connected Devices," or "ARP Table"** section. This table will show a list of all devices connected to your network, along with their IP addresses and MAC addresses. Look for entries that correspond to your camera's MAC address (often found on a sticker on the camera itself).
        * *(For Airtel routers, this is typically `192.168.1.1`; for Jio, it's often `192.168.29.1`.)*
    * **Assign Static IPs (Recommended):**
        Once you've identified your camera's IP, it's highly recommended to assign it a **static IP address**. This prevents the camera's IP from changing, which would break your Frigate configuration. You can do this in two ways:
        1.  **Via Camera's Web Interface:** Most IP cameras have their own web interface. Navigate to `http://CAMERA_IP_ADDRESS` (e.g., `http://192.168.5.19`) in your browser. Log in and find the network settings. Change the DHCP setting to "Static IP" and manually enter an IP address, subnet mask, gateway, and DNS server. **Ensure this static IP is outside your router's DHCP range to prevent conflicts.**
        2.  **DHCP Reservation in Router:** Even better, you can often "reserve" an IP address for your camera's MAC address directly in your router's DHCP settings. This means the camera still requests an IP via DHCP, but your router will *always* assign it the same static IP.

2.  **Access Camera Web Interface & Set Passwords:**
    Navigate to each camera's IP address in your web browser (e.g., `http://192.168.5.19`). You'll be prompted for a username and password.

    * **Existing Cameras:** If the camera was previously used with an NVR, the password might be the same as your NVR's password.
    * **New Cameras:** New cameras typically have a default password (e.g., `admin/admin`, `admin/12345`, `root/pass`) or no password at all, requiring you to set one on first access.
    * **Important:** Even if the live feed doesn't load due to plugin issues in your browser (common for older cameras), you can still access the settings.

3.  **Find Camera RTSP URLs using Nmap:**
    To get the live stream from your camera into Frigate, you need its RTSP (Real-Time Streaming Protocol) URL. Many cameras don't publish these publicly, so you can discover them using `nmap`.

    * **Install Nmap:** On any Linux machine on your network (your Proxmox host, a desktop, or even another LXC), install `nmap` if you haven't already:
        ```bash
        sudo apt update && sudo apt install nmap -y
        ```
    * **Brute-Force RTSP URLs:** Run `nmap` with the `rtsp-url-brute` script, replacing `192.168.5.19` with your camera's actual IP address. This script attempts common RTSP paths.
        ```bash
        sudo nmap --script rtsp-url-brute -p 554 192.168.5.19
        sudo nmap --script rtsp-url-brute -p 8554 192.168.5.19
        ```
        The output will list potential RTSP URLs. Look for URLs that start with `rtsp://` and contain `/stream` or `/channel` paths.

4.  **Verify RTSP Streams with VLC:**
    Once you have a potential RTSP URL, verify it by opening it in VLC Media Player.

    * Open VLC.
    * Go to `Media` > `Open Network Stream...`
    * Paste the full RTSP URL. Remember to include your camera's **username and password** directly in the URL:
        * **Generic/CP Plus Example:** `rtsp://<username>:<password>@<camera_ip>/cam/realmonitor?channel=1&subtype=00`
        * **Hikvision Example:** `rtsp://<username>:<password>@<camera_ip>:554/stream2`
        * *(Note: The exact paths (`/cam/realmonitor`, `/stream2`, `/Streaming/Channels/101`, etc.) vary significantly by camera brand and model. You might need to try different ones found by `nmap` or search online for your specific camera model's RTSP URL format.)*

    If you get a live view, congratulations! You have successfully identified the correct RTSP URL for your camera. **Repeat these steps for all your cameras and carefully note down each camera's static IP, username, password, and working RTSP URL.**

---

# Frigate NVR: Camera Configuration Guide

This section will guide you through the process of connecting your IP cameras to your network, finding their RTSP streams, and configuring them within Frigate.

## Part 1: Discovering and Securing Your IP Cameras

Before adding cameras to Frigate, you need to ensure they are accessible on your local network and that you have their RTSP stream URLs.

1.  **Connect Cameras to Your Network & Assign Static IPs:**
    Connect all your IP cameras to your network router/switch. If your cameras were previously configured with an NVR, they might already have an assigned IP address.

    - **Find Camera IPs:**
      - Log in to your **router's web interface**.
      - Navigate to the **"Client List," "Connected Devices," or "ARP Table"** section. This table will show a list of all devices connected to your network, along with their IP addresses and MAC addresses. Look for entries that correspond to your camera's MAC address (often found on a sticker on the camera itself).
      - _(For Airtel routers, this is typically `192.168.1.1`; for Jio, it's often `192.168.29.1`.)_
    - **Assign Static IPs (Recommended):**
      Once you've identified your camera's IP, it's highly recommended to assign it a **static IP address**. This prevents the camera's IP from changing, which would break your Frigate configuration. You can do this in two ways:
      1.  **Via Camera's Web Interface:** Most IP cameras have their own web interface. Navigate to `http://CAMERA_IP_ADDRESS` (e.g., `http://192.168.5.19`) in your browser. Log in and find the network settings. Change the DHCP setting to "Static IP" and manually enter an IP address, subnet mask, gateway, and DNS server. **Ensure this static IP is outside your router's DHCP range to prevent conflicts.**
      2.  **DHCP Reservation in Router:** Even better, you can often "reserve" an IP address for your camera's MAC address directly in your router's DHCP settings. This means the camera still requests an IP via DHCP, but your router will _always_ assign it the same static IP.

2.  **Access Camera Web Interface & Set Passwords:**
    Navigate to each camera's IP address in your web browser (e.g., `http://192.168.5.19`). You'll be prompted for a username and password.
    ![alt text](<../setup/screenshots/Screenshot 2025-05-22 at 5.03.31 PM.jpg>)
    ![alt text](<../setup/screenshots/Screenshot 2025-05-22 at 5.04.36 PM.jpg>)

    - **Existing Cameras:** If the camera was previously used with an NVR, the password might be the same as your NVR's password.
    - **New Cameras:** New cameras typically have a default password (e.g., `admin/admin`, `admin/12345`, `root/pass`) or no password at all, requiring you to set one on first access.
    - **Important:** Even if the live feed doesn't load due to plugin issues in your browser (common for older cameras), you can still access the settings.

3.  **Find Camera RTSP URLs using Nmap:**
    To get the live stream from your camera into Frigate, you need its RTSP (Real-Time Streaming Protocol) URL. Many cameras don't publish these publicly, so you can discover them using `nmap`.

    - **Install Nmap:** On any Linux machine on your network (your Proxmox host, a desktop, or even another LXC), install `nmap` if you haven't already:
      ```bash
      sudo apt update && sudo apt install nmap -y
      ```
    - **Brute-Force RTSP URLs:** Run `nmap` with the `rtsp-url-brute` script, replacing `192.168.5.19` with your camera's actual IP address. This script attempts common RTSP paths.
      ```bash
      sudo nmap --script rtsp-url-brute -p 554 192.168.5.19
      sudo nmap --script rtsp-url-brute -p 8554 192.168.5.19
      ```
      The output will list potential RTSP URLs. Look for URLs that start with `rtsp://` and contain `/stream` or `/channel` paths.

4.  **Verify RTSP Streams with VLC:**
    Once you have a potential RTSP URL, verify it by opening it in VLC Media Player.

    - Open VLC.
    - Go to `Media` > `Open Network Stream...`
    - Paste the full RTSP URL. Remember to include your camera's **username and password** directly in the URL:
      - **Generic/CP Plus Example:** `rtsp://<username>:<password>@<camera_ip>/cam/realmonitor?channel=1&subtype=00`
      - **Hikvision Example:** `rtsp://<username>:<password>@<camera_ip>:554/stream2`
      - _(Note: The exact paths (`/cam/realmonitor`, `/stream2`, `/Streaming/Channels/101`, etc.) vary significantly by camera brand and model. You might need to try different ones found by `nmap` or search online for your specific camera model's RTSP URL format.)_

    **Repeat these steps for all your cameras and carefully note down each camera's static IP, username, password, and working RTSP URL.**

---

## Part 2: Configuring Frigate's `config.yml`

Now that you have your camera stream details, you can configure Frigate to process them.

1.  **Access Frigate LXC & Navigate to Config Directory:**
    Log back into your Frigate LXC (e.g., via SSH).
    Navigate to the `config` directory within your Frigate installation:

    ```bash
    cd ~/Downloads/programs/frigate/config
    ```

2.  **Create/Edit `config.yml`:**
    Frigate uses a `config.yml` file for all its settings. Create this file if it doesn't exist, or edit it if you have a default one (like `config.yml.template`).

    ```bash
    vi config.yml # Or nano config.yml if you prefer nano
    ```

    Press `i` (for `vi`) to enter insert mode, then paste the following configuration.

    ```yaml
    # General Frigate configuration
    mqtt:
      host: YOUR_MQTT_BROKER_IP # e.g., IP of Home Assistant or dedicated MQTT server
      user: YOUR_MQTT_USERNAME
      password: YOUR_MQTT_PASSWORD
    detectors:
      coral:
        type: edgetpu
        device: usb # Or 'pci' if using a PCIe Coral. Use 'usb:1' for second USB Coral, etc.

    # Camera Configurations
    cameras:
      front_door_closeup:
        ffmpeg:
          inputs:
            # Main stream for recording (usually higher resolution)
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.101:554/Streaming/Channels/101
              roles:
                - record
            # Sub-stream for detection (usually lower resolution for efficiency)
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.101:554/Streaming/Channels/102
              roles:
                - detect
          # Output arguments for FFmpeg (for recording quality/format)
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640 # Resolution of the detect stream
          height: 360
          fps: 20 # Frames per second for detection
        objects:
          track:
            - person
            - car
            - motorcycle
            # ... and other objects you want to track from Frigate's supported list
            - bird
            - cat
            - dog
            - horse
            - sheep
            - cow
            - bear
            - zebra
            - giraffe
            - elephant
            - mouse
          filters:
            person:
              # Example: mask out areas where persons should not be detected
              mask: 570,299,545,0 # Coordinates for a polygon mask
            cat:
              min_score: 0.01
              threshold: 0.02
            # ... apply filters for other objects as needed (adjust sensitivity)
        motion:
          # Optional: Mask out areas to ignore motion detection
          mask:
            - 473,0,21,156,53,317,140,312 # Example polygon mask
        record:
          enabled: true
          events:
            pre_capture: 5 # Seconds of video before event
            post_capture: 5 # Seconds of video after event
            objects:
              # Objects that trigger an event recording
              - person
              - car
              - motorcycle
              # ... other objects as desired

      driveway:
        # Repeat the ffmpeg, detect, objects, motion, and record sections for each camera
        ffmpeg:
          inputs:
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.102:554/Streaming/Channels/101
              roles:
                - record
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.102:554/Streaming/Channels/102
              roles:
                - detect
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640
          height: 360
          fps: 20
        objects:
          track:
            - person
            - car
            - motorcycle
            - bird
            - cat
            - dog
            - horse
            - sheep
            - cow
            - bear
            - zebra
            - giraffe
            - elephant
            - mouse
          filters:
            car:
              min_score: 0.01
              threshold: 0.03
            cat:
              min_score: 0.01
              threshold: 0.02
            dog:
              min_score: 0.01
              threshold: 0.02
            bird:
              min_score: 0.01
              threshold: 0.02
        record:
          enabled: true
          events:
            pre_capture: 5
            post_capture: 5
            objects:
              - person
              - car
              - motorcycle
              - bird
              - cat
              - dog
              - horse
              - sheep
              - cow
              - bear
              - zebra
              - giraffe
              - elephant
              - mouse

      side_door_closeup:
        ffmpeg:
          inputs:
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.104:554/Streaming/Channels/101
              roles:
                - record
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.104:554/Streaming/Channels/102
              roles:
                - detect
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640
          height: 360
          fps: 20
        objects:
          track:
            - person
            - bird
            - cat
            - dog
            - horse
            - sheep
            - cow
            - bear
            - zebra
            - giraffe
            - elephant
            - mouse
          filters:
            car:
              min_score: 0.01
              threshold: 0.03
            cat:
              min_score: 0.01
              threshold: 0.02
            dog:
              min_score: 0.01
              threshold: 0.02
            bird:
              min_score: 0.70
              threshold: 0.75
        record:
          enabled: true
          events:
            pre_capture: 5
            post_capture: 5
            objects:
              - person
              - car
              - bird
              - cat
              - dog
              - horse
              - sheep
              - cow
              - bear
              - zebra
              - giraffe
              - elephant
              - mouse

      back_door_closeup:
        ffmpeg:
          inputs:
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.103:554/Streaming/Channels/101
              roles:
                - record
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.103:554/Streaming/Channels/102
              roles:
                - detect
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640
          height: 360
          fps: 20
        objects:
          track:
            - person
            - car
            - bird
            - cat
            - dog
            - horse
            - sheep
            - cow
            - bear
            - zebra
            - giraffe
            - elephant
            - mouse
          filters:
            car:
              min_score: 0.75
              threshold: 0.75
            cat:
              min_score: 0.01
              threshold: 0.02
            dog:
              min_score: 0.01
              threshold: 0.02
            bird:
              min_score: 0.01
              threshold: 0.02
        record:
          enabled: true
          events:
            pre_capture: 5
            post_capture: 5
            objects:
              - person
              - car
              - bird
              - cat
              - dog
              - horse
              - sheep
              - cow
              - bear
              - zebra
              - giraffe
              - elephant
              - mouse

      front_porch_wide_angle:
        ffmpeg:
          inputs:
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.106:554/Streaming/Channels/101
              roles:
                - record
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.106:554/Streaming/Channels/102
              roles:
                - detect
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640
          height: 360
          fps: 20
        objects:
          track:
            - person
            - car
            - motorcycle
            - bird
            - cat
            - dog
            - horse
            - sheep
            - cow
            - bear
            - zebra
            - giraffe
            - elephant
            - mouse
          filters:
            person:
              min_score: 0.8
              threshold: 0.8
            car:
              min_score: 0.6
              threshold: 0.7
            cat:
              min_score: 0.01
              threshold: 0.02
            dog:
              min_score: 0.01
              threshold: 0.02
            bird:
              min_score: 0.6
              threshold: 0.65
        record:
          enabled: true
          events:
            pre_capture: 5
            post_capture: 5
            objects:
              - person
              - car
              - motorcycle
              - bird
              - cat
              - dog
              - horse
              - sheep
              - cow
              - bear
              - zebra
              - giraffe
              - elephant
              - mouse

      fishcam:
        ffmpeg:
          inputs:
            # Example using a different stream for detect than record
            - path: rtsp://YOUR_CAMERA_USERNAME:YOUR_CAMERA_PASSWORD@192.168.3.120:554/stream1
              roles:
                - record
                - detect # Both roles can use the same stream if needed
          output_args:
            record: -f segment -segment_time 60 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy
        detect:
          width: 640
          height: 360
          fps: 20
        objects:
          track:
            - person # Even if it's a fishcam, you might want to detect people walking by
            - fish # Custom object for fishcam (requires custom model training, not default)
          filters:
            person:
              min_score: 0.3
              threshold: 0.3
        record:
          enabled: true
          events:
            pre_capture: 15
            post_capture: 15
            objects:
              - fish # Only record if 'fish' is detected (requires custom model)

    database:
      path: /data/db/frigate.db # Path to the Frigate SQLite database within the container
    ```

    **Config credit `https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_&_28_minute_presentation_by_FUTO_software#Alternatives_for_the_Best_Quality`**

    **Important Modifications to the `config.yml`:**

    - **`mqtt` section:** You **MUST** fill in your MQTT broker details here. Frigate relies heavily on MQTT for communication, especially with Home Assistant. If you have Home Assistant, use its MQTT broker IP, username, and password. Otherwise, you'll need to set up a separate MQTT broker (e.g., Mosquitto).
    - **`detectors` section:**
      - `device: usb` or `device: pci`. Choose based on how your Coral TPU is connected. If you have multiple USB Corals, you can specify `usb:0`, `usb:1`, etc.
    - **`cameras` section:**

      - **`YOUR_CAMERA_USERNAME` and `YOUR_CAMERA_PASSWORD`:** Replace these placeholders with the actual username and password for _each specific camera_.
      - **`192.168.3.x`:** Replace these IP addresses with the **static IP addresses** you assigned to your cameras.
      - **`path`:** Update these RTSP URLs with the exact ones you verified using VLC for each camera. Pay attention to the stream path (e.g., `/Streaming/Channels/101`, `/stream1`, `/video.cgi`).
      - **Roles (`record` vs. `detect`):** It's common practice to use a higher-resolution stream for `record` (e.g., your camera's main stream) and a lower-resolution sub-stream for `detect` (more efficient for AI processing). Check your camera's capabilities for multiple streams.
      - **`objects` & `filters`:** Adjust the `track` list to include only the objects you care about. `min_score` and `threshold` allow you to fine-tune detection sensitivity. Masks (e.g., `mask: 570,299,545,0`) are incredibly useful to ignore areas in the camera's view where you don't want detection (e.g., public sidewalks, trees moving in the wind).

    - **Save and Exit:** If using `vi`, press `Esc`, then type `:wq` and press `Enter`. If using `nano`, press `Ctrl+X`, then `Y` to save, and `Enter`.

3.  **Clean Up (Optional):**
    If the `config` directory contains a `demo-config.yml` or similar sample file, you can remove it to avoid confusion:

    ```bash
    rm demo-config.yml # If it exists
    ```

4.  **Navigate Back to Frigate's Main Directory:**

    ```bash
    cd ..
    ```

    You should now be in the `~/Downloads/programs/frigate` directory, where your `docker-compose.yml` resides.

    ```bash
    root@docker:/home/root/Downloads/programs/frigate# ls
    audio-labelmap.txt   cspell.json        frigate         migrations       pyproject.toml
    benchmark_motion.py  database           go2rtc.yaml     netlify.toml     README_CN.md
    benchmark.py         docker             labelmap.txt    notebooks        README.md
    CODEOWNERS           docker-compose.yml LICENSE         package-lock.json  storage
    config               docs               Makefile        process_clip.py    web
    root@docker:/home/root/Downloads/programs/frigate#
    ```

5.  **Start Frigate Services:**
    Finally, start Frigate and its associated services using Docker Compose:

    ```bash
    docker compose up -d
    ```

    This command will build (if necessary) and start all the containers defined in your `docker-compose.yml` file in detached mode (meaning they run in the background).

6.  **Access Frigate Web Interface:**
    Open your web browser and navigate to the IP address of your Frigate LXC on port 5000 (or 8971 if you only mapped 8971):
    `http://YOUR_FRIGATE_LXC_IP:5000`

    You should now see the Frigate web interface, showing your camera feeds, object detections, and events!

---
