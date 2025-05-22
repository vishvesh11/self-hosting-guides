---
# My Homelab & Self-Hosting Guides

This repository serves as a comprehensive documentation of my personal homelab setup and various self-hosted services. It's a living knowledge base detailing the architecture, configuration, and troubleshooting steps for the infrastructure I manage.

This project showcases my hands-on experience in system administration, networking, containerization, security, and automation, reflecting a passion for building, maintaining, and optimizing IT environments.
---

## 🚀 What's Inside?

This repository contains step-by-step guides, configuration snippets, and insights into setting up and managing a robust home server environment. You'll find documentation on:

- **Infrastructure Management:** Proxmox VE, ZFS storage configurations, and network architecture with pfSense.
- **Core Network Services:** Detailed setup for self-hosting email (Mailcow)\*, advanced VPN solutions (WireGuard, Tailscale), and centralized access dashboards (Homarr).
- **Application Hosting:** Deploying and managing various applications using Docker/Docker Compose, including Nginx Proxy Manager, Immich (photo management), Jellyfin (media server), and qBittorrent.
- **Home Automation & AI:** Configuring Home Assistant for smart home control and integrating AI-powered surveillance with Frigate and Google Coral TPU.
- **Security & Monitoring:** Strategies for securing services, managing SSL/TLS certificates (Let's Encrypt), and proactive system monitoring (Uptime Kuma).
- **Data Management:** Implementing robust backup strategies and file synchronization with Syncthing.

Each guide aims to provide practical, actionable steps and highlight key challenges and solutions encountered during implementation.

---

## 🛠️ Technologies & Skills required

Through this homelab, I got practical experience with a wide array of technologies and principles, including:

- **Operating Systems:** Debian/Ubuntu Linux
- **Virtualization & Containerization:** Proxmox VE, LXC, Docker, Docker Compose
- **Networking:** pfSense, DNS, DHCP, VPN (WireGuard, Tailscale), VLANs, Firewall Rules, Reverse Proxies (Nginx Proxy Manager), Cloudflare Tunnels
- **Storage:** ZFS, RAID 1, RAID 5, RAID 6
- **Databases:** PostgreSQL, MySQL/MariaDB (through various applications)
- **Security:** SSL/TLS (Let's Encrypt), SSH, Vaultwarden, Network Security Best Practices
- **Monitoring & Logging:** Uptime Kuma, System Monitoring Tools
- **Automation & Scripting:** Bash Scripting, Home Assistant Automations (YAML)
- **Applications:** Mailcow, Immich, Jellyfin, Home Assistant, Frigate, Syncthing, qBittorrent, Homarr, Excalidraw, Vaultwarden, Audiobookshelf

---

## 🗺️ Repository Structure

```
.
├── README.md                           <- You are here!
├── setup/                              <- Guides for core infrastructure setup (Proxmox, ZFS, pfSense)
│   ├── proxmox_installation.md
│   ├── zfs_raid1_config.md
│   └── pfsense_initial_config.md
│   └── network_overview.md
├── services/                           <- Documentation for individual self-hosted applications
│   ├── mailcow_email_server.md
│   ├── nginx_proxy_manager_setup.md
│   ├── wireguard_vpn_server.md
│   ├── vaultwarden_password_manager.md
│   ├── cloudflare_tunnels_config.md
│   ├── immich_photo_archive.md (Immich setup, esp. with reverse proxy)
│   ├── jellyfin_media_server.md
│   ├── homeassistant_automations.md
│   ├── frigate_with_coral_tpu.md (google coral m.2 A+E key )
│   ├── syncthing_file_sync.md
│   ├── qbitorrent_setup.md
│   └── homarr_dashboard_config.md
├── security/                           <- General security hardening and best practices
│   └── general_security_hardening.md
│   ├── ssl_tls_with_letsencrypt.md

├── automation/                         <- Scripts and methods for automation and backups
│   └── backup_strategies.md
└── troubleshooting/                    <- Common issues and their resolutions
    └── common_issues_and_fixes.md
```

---

## 💡 Why a Homelab?

My homelab serves as a personal sandbox for continuous learning, experimentation and gives me opportunity to make something my family can use. It allows us to:

- **Gain hands-on experience** with enterprise-grade and open-source technologies in a real-world environment.
- **Develop problem-solving skills** by troubleshooting complex interactions between hardware, software, and network components.
- **Explore new technologies** and understand their practical applications in a controlled setting.
- **Build a resilient and private personal infrastructure**, taking control of my data and services.

---

## 🔗 Connect With Me

- **Portfolio:** vishvesh.me
- **Email:** vishveshspal@gmail.com ; mail@vishvesh.me
- **LinkedIn:** https://www.linkedin.com/in/vishvesh-singh-pal-13aba5157/

---

Feel free to explore the guides, and don't hesitate to reach out if you have any questions or feedback\!

---
