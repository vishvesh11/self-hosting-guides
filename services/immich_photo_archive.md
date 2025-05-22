---

# Immich Photo & Video Archive Setup Guide

This guide details the process of setting up Immich, a self-hosted photo and video backup solution, using Docker Compose on a Proxmox VM or LXC. Immich is an open-source alternative to Google Photos, allowing you to retain full control over your media.

## Prerequisites:

* **Proxmox VE Installed:** Your Proxmox host is up and running.
* **Dedicated VM or LXC for Docker:** A virtual machine (VM) or Linux Container (LXC) on your Proxmox server specifically for running Docker containers.
    * **Recommended OS for VM/LXC:** Debian.
    * **Resources:** Allocate at least 4GB RAM and 2-4 CPU cores to this VM/LXC, especially if you plan to enable machine learning features in Immich later or have a large library.
* **Internet Access:** The VM/LXC needs internet access to download packages and Docker images.

## Part 1: Setting up Docker on Your VM/LXC

The first step is to get Docker and Docker Compose installed.

### Option A: Using a Proxmox Community Helper Script (Recommended: easy)


1.  **Access Proxmox Host Shell:** Open a shell to your Proxmox VE host (e.g., via the Proxmox web interface `>_ Shell` icon for your `pve` node, or SSH into `root@YOUR_PROXMOX_IP`).
2.  **Download and Run Docker VM/LXC Helper Script:**
    Visit the [Proxmox Community Scripts GitHub page](https://community-scripts.github.io/ProxmoxVE/scripts?id=docker-vm) for the latest script.
    Typically, you'd run something like:
    ```bash
    bash -c "$(wget -qLO - https://raw.githubusercontent.com/tteck/Proxmox/main/ct/docker.sh)"
    ```
    Follow the on-screen prompts to create your Docker VM or LXC. This script automates Docker and Docker Compose installation.

### Option B: Manual Docker Installation (If you prefer manual control)

If you already have a VM/LXC or prefer to install Docker manually, follow these steps.

1.  **Connect to Your VM/LXC:**
    SSH into your newly created (or existing) VM/LXC. Replace `your-user` and `YOUR_VM/LXC_IP` with your actual credentials.
    ```bash
    ssh your-user@YOUR_VM_LXC_IP
    ```

2.  **Update System Packages:**
    Ensure your VM/LXC's package lists are up-to-date and upgrade any existing packages.
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

3.  **Install Docker Engine:**
    Follow the official Docker installation guide for your specific Linux distribution (e.g., Ubuntu, Debian).
    **For Ubuntu/Debian (common choice):**
    ```bash
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    # Install Docker Engine, containerd, and Docker Compose plugin
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

4.  **Enable and Start Docker Service:**
    Ensure the Docker service starts automatically on boot and is currently running.
    ```bash
    sudo systemctl enable --now docker
    ```

5.  **Add User to Docker Group (Optional but Recommended):**
    To run Docker commands without `sudo`, add your user to the `docker` group. **You'll need to log out and log back in for this to take effect.**
    ```bash
    sudo usermod -aG docker $USER
    ```
    After running this, log out of your SSH session and log back in. Test if Docker works without `sudo`:
    ```bash
    docker run hello-world
    ```
    If it runs successfully, you're good to go!

## Part 2: Installing Immich with the Official Script

Immich provides a convenient installer script that sets up all the necessary Docker containers and configurations.

1.  **Create an Immich Directory:**
    Navigate to your home directory or create a dedicated directory for Immich.
    ```bash
    cd ~
    mkdir immich && cd immich
    ```

2.  **Run the Official Immich Installation Script:**
    This script will download the `docker-compose.yml` file and other necessary configuration files, then start the Immich services.
    ```bash
    curl -o- https://raw.githubusercontent.com/immich-app/immich/main/install.sh | bash
    ```
    * **Follow Prompts:** The script will ask you some questions, such as where to store your Immich data (e.g., `/mnt/data/immich`). **Ensure you choose a path on a drive with sufficient space for your photos and videos.** This is typically a mounted network share or a large disk passed through to your VM.
    * **GPU Transcoding:** If your Proxmox host has a compatible GPU and you've passed it through to this VM, the script might ask about GPU transcoding. Enable this for faster video processing if you have the hardware.

3.  **Wait for Services to Start:**
    The script will pull all necessary Docker images and start the containers. This process can take several minutes, especially on the first run, as it downloads large image files.

## Part 3: Initial Access and Post-Installation

1.  **Access Immich Web Interface:**
    Once the script finishes and services are running, Immich should be accessible via your browser.
    Open a web browser and navigate to:
    `http://YOUR_VM_LXC_IP:2283`
    (Replace `YOUR_VM_LXC_IP` with the actual IP address of your Docker VM/LXC).

2.  **Create Admin Account:**
    The first time you access Immich, you'll be prompted to create an administrator account. Follow the on-screen instructions.


## Part 4: Next Steps - Security and External Access

For a production-ready and secure Immich setup, you need to:

* **Configure OAuth / Single Sign-On (SSO):** This is highly recommended for secure user management.
* **Set up a Reverse Proxy (Nginx Proxy Manager):** To expose Immich to the internet securely using a custom domain and SSL/TLS certificates.
* **Expose to Internet with Cloudflare Tunnels:** (As per your plan) for secure external access without opening ports.

**Please refer to the `security/nginx_proxy_manager_setup.md` and `services/cloudflare_tunnels_config.md` guides in this repository for detailed instructions on securing and exposing your Immich instance to the internet.**

---
