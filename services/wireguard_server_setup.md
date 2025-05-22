---

# WireGuard VPN Server Setup: Home Network Gateway

This guide will walk you through setting up a WireGuard VPN server on a Virtual Private Server (VPS) that acts as a secure gateway, allowing external clients to connect to your home network. This setup is ideal for accessing your self-hosted services or devices securely from outside your home.

## Understanding the Network Topology

Your WireGuard tunnel will follow a hub-and-spoke model:

**Outside Client <---> VPS (WireGuard Server/Gateway) <---> Home Network**

* **VPS:** This will be your central WireGuard server. It will have a public IP address and act as the entry point for all your VPN clients. It will also be configured to forward traffic to your home network.
* **Home Network:** Your home router/firewall (like pfSense) will act as a WireGuard client, establishing a persistent tunnel to the VPS. This allows the VPS to route traffic directly into your home network.


---

## Part 1: Setting up the WireGuard Server on your VPS

### Step 1: Install WireGuard on the VPS

Log in to your VPS via SSH. First, update your system and then install WireGuard.

**For Debian/Ubuntu:**

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install wireguard qrencode -y # qrencode is for generating QR codes for clients
```

### Step 2: Generate Server Keys

Generate the private and public keys for your WireGuard server.

```bash
wg genkey | sudo tee /etc/wireguard/privatekey_server
sudo chmod 600 /etc/wireguard/privatekey_server # Restrict permissions
sudo cat /etc/wireguard/privatekey_server | wg pubkey | sudo tee /etc/wireguard/publickey_server
```

Now, view your generated keys:

```bash
echo "Server Private Key:" && sudo cat /etc/wireguard/privatekey_server
echo "Server Public Key:" && sudo cat /etc/wireguard/publickey_server
```

**Note down your Server Private Key and Server Public Key.** You'll need these shortly.

### Step 3: Configure the VPS WireGuard (`wg0.conf`)

Now, create the WireGuard configuration file for your VPS. This will be `/etc/wireguard/wg0.conf`.

```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste the following configuration into the file. **Remember to replace the placeholders** with your actual values:

- `<VPS_PRIVATE_KEY>`: The server private key you generated above.
- `<HOME_NODE_PUBLIC_KEY>`: This will be the public key of your home router/firewall's WireGuard client (you'll generate this on your home router later).
- `<HOME_NODE_ALLOWED_IP>`: This is the **WireGuard tunnel IP** you'll assign to your home router (e.g., `10.69.69.2`).
- `<HOME_NETWORK_SUBNET>`: Your home network's actual subnet (e.g., `192.168.5.0/24`). This tells the VPS to route traffic for your entire home network through the tunnel to your home router.
- `<VPS_WAN_INTERFACE>`: The name of your VPS's public-facing network interface (e.g., `eth0`, `ens3`, `enp0s3`). You can find this by running `ip a` on your VPS.

```ini
# /etc/wireguard/wg0.conf on the VPS

[Interface]
PrivateKey = <VPS_PRIVATE_KEY>
Address = 10.69.69.1/24 # This is the WireGuard IP for the VPS itself within the tunnel
ListenPort = 51820

# Enable IP forwarding and NAT for internet access and routing to home network
PostUp = iptables -t nat -A POSTROUTING -o <VPS_WAN_INTERFACE> -j MASQUERADE
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT

PostDown = iptables -t nat -D POSTROUTING -o <VPS_WAN_INTERFACE> -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT

[Peer] # Home Network Router/Firewall
PublicKey = <HOME_NODE_PUBLIC_KEY>
AllowedIPs = <HOME_NODE_ALLOWED_IP>/32, <HOME_NETWORK_SUBNET>
PersistentKeepalive = 25

# Below peers will be added by the script for your mobile clients
# [Peer] # Phone Client 1
# PublicKey = ...
# AllowedIPs = 10.69.69.X/32
# PersistentKeepalive = 25
```

Save and exit the file (Ctrl+X, Y, Enter for Nano; Esc, :wq, Enter for Vi).

### Step 4: Enable IP Forwarding on VPS

For the VPS to act as a router and forward traffic between your WireGuard tunnel and the internet/home network, you must enable IP forwarding.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1 # If you plan to use IPv6
```

To make this persistent across reboots, edit `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add the following lines:

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Save and exit. Apply the changes:

```bash
sudo sysctl -p
```

### Step 5: Start WireGuard Service on VPS

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Verify the status:

```bash
sudo systemctl status wg-quick@wg0
sudo wg show
```

You should see `wg0` interface active, but no peers yet.

---

## Part 2: Setting up WireGuard on Your Home Router/Firewall

This section assumes you are using a router/firewall that supports WireGuard clients (like pfSense, OpenWrt, or a compatible consumer router). The steps will vary slightly depending on your specific device.

### Step 1: Generate Keys for Home Node

On your **home router/firewall**, generate a private and public key for its WireGuard client.

- **For pfSense/OpenWrt:** Navigate to the WireGuard configuration section in the web UI. There should be an option to generate new keys.
- **If you have a Linux machine on your home network:** You can generate them there:
  ```bash
  wg genkey | tee privatekey_home
  cat privatekey_home | wg pubkey | tee publickey_home
  ```
  **Note down your Home Node Private Key and Home Node Public Key.**

### Step 2: Configure Home Node WireGuard Client

On your **home router/firewall**, configure its WireGuard client. This usually involves a UI in pfSense/OpenWrt.

- **Interface (Local) Settings:**

  - **Private Key:** The `Home Node Private Key` you just generated.
  - **Address:** This is the WireGuard tunnel IP for your home router (e.g., `10.69.69.2/32`). Use a `/32` CIDR as it's a specific host in the tunnel.
  - **Listen Port:** (Optional) You can leave this blank or use a random high port, as your home router will be initiating the connection.

- **Peer (VPS Server) Settings:**
  - **Public Key:** The `Server Public Key` you generated on your VPS.
  - **Endpoint:** The public IP address of your VPS and the WireGuard port: `<VPS_PUBLIC_IP>:51820`.
  - **Allowed IPs:** `10.69.69.0/24` (This is the entire WireGuard tunnel subnet, allowing your home router to reach all VPN clients) and `0.0.0.0/0` (This sends _all_ traffic from your home router through the VPN tunnel to the VPS, effectively backhauling your entire home network's internet traffic through the VPS. **Use `0.0.0.0/0` only if you want all your home network's internet traffic to exit via the VPS's public IP.** If you _only_ want to access your VPS and other VPN clients, use `10.69.69.0/24`).
  - **Persistent Keepalive:** `25` seconds (Helps maintain the connection through NAT).

### Step 3: Configure Firewall Rules on Home Router/Firewall

Ensure your home router's firewall allows the WireGuard traffic to pass through.

- **Outbound Rule:** Allow UDP traffic from your home router's WAN interface to `VPS_PUBLIC_IP` on port `51820`. (Often, initiating the connection from inside is enough, but explicit rules can help).
- **Inbound Rule on WireGuard Interface:** Allow traffic on the newly created WireGuard interface to pass to your home network. You might need rules to allow `10.69.69.0/24` access to your `192.168.5.0/24` network.

Apply changes on your home router and try to bring up the WireGuard tunnel.

### Step 4: Verify Home Node Connection to VPS

On your VPS, run `sudo wg show` again. You should now see the `Home Network Router` peer listed, along with its allowed IP and a "latest handshake" timestamp if the connection is successful.

```bash
sudo wg show
```

---

## Part 3: Generating Client Configurations on VPS

Now that your VPS is acting as a WireGuard server and is connected to your home network, you can generate configurations for your mobile clients (phones, laptops). The provided script automates this process.

### Step 1: Install `git` and `qrencode` on VPS (if not already)

Ensure `git` and `qrencode` are installed on your VPS. `qrencode` is used by the script to generate QR codes for easy client setup.

```bash
sudo apt install git qrencode -y # Debian/Ubuntu
# sudo yum install git qrencode -y # CentOS/RHEL
```

### Step 2: Prepare the Script and Output Directory

1.  Create a directory on your VPS to store the script and its output:
    ```bash
    mkdir -p ~/wireguard_client_gen
    cd ~/wireguard_client_gen
    ```
2.  Create the `generate_wireguard.sh` file:
    ```bash
    nano generate_wireguard.sh
    ```
3.  Paste the provided script into this file:

    ```bash
    #!/usr/bin/env bash
    #
    # generate_wireguard.sh
    #
    # Example script to generate:
    #   - Server config (wg0.conf) with 10 phone peers (for addition to VPS wg0.conf)
    #   - 10 phone configs (phone1.conf, phone2.conf, ...)
    #   - Print 10 QR codes in the terminal for easy scanning
    #
    # Requirements: wg, qrencode


    # 1) EDIT THESE VARIABLES FOR YOUR ENVIRONMENT


    # The WireGuard servers private key (already generated).
    # You MUST replace this with your actual VPS servers private key.
    SERVER_PRIVATE_KEY="<YOUR_VPS_SERVER_PRIVATE_KEY_HERE>" # Replace this!

    # The servers public key is derived from the private key:
    SERVER_PUBLIC_KEY="$(echo "$SERVER_PRIVATE_KEY" | wg pubkey)" # This will be derived automatically

    # The public (internet-facing) IP or domain of your WireGuard server:
    SERVER_PUBLIC_IP="<YOUR_VPS_PUBLIC_IP_OR_DOMAIN>" # IMPORTANT: Replace this!

    # The UDP listen port:
    SERVER_PORT="51820"

    # The interface (on the server) that goes to the Internet (for MASQUERADE rules):
    SERVER_WAN_IFACE="<YOUR_VPS_WAN_INTERFACE>" # IMPORTANT: Replace this! (e.g., eth0, ens3)

    # The WireGuard interface name on the server:
    WG_INTERFACE="wg0"

    # The server's internal tunnel IP (we'll assume /24):
    SERVER_TUN_IP="10.69.69.1" # This should match the Address in your VPS wg0.conf

    # Range of client IPs. We'll generate 10 clients, from .10 to .19 in the 10.69.69.x subnet
    START_IP=10 # 10.69.69.10
    END_IP=19   # 10.69.69.19

    # Where to store the final server config and client configs:
    OUTPUT_DIR="./output"
    SERVER_CONFIG_FRAGMENT="$OUTPUT_DIR/server_peers_fragment.conf" # This file will contain only the [Peer] blocks for clients


    # 2) PREPARE THE OUTPUT DIRECTORY

    mkdir -p "$OUTPUT_DIR"
    rm -f "$SERVER_CONFIG_FRAGMENT" # start fresh


    # 3) BUILD THE SERVER'S [Interface] SECTION (Already done in Part 1)
    #    This script will only generate the [Peer] blocks for clients
    #    You will manually append these to your existing /etc/wireguard/wg0.conf



    # 4) LOOP OVER CLIENTS AND BUILD:
    #    - A [Peer] entry for the server config fragment
    #    - A separate clientX.conf file
    #    - Print the client's QR code


    i=1
    for IP_OFFSET in $(seq $START_IP $END_IP); do
      CLIENT_IP="10.69.69.$IP_OFFSET"
      CLIENT_PRIV_KEY="$(wg genkey)"
      CLIENT_PUB_KEY="$(echo "$CLIENT_PRIV_KEY" | wg pubkey)"

      # -- BUILD CLIENT CONF FILE --
      CLIENT_CONF="$OUTPUT_DIR/client${i}.conf"
      cat <<EOCLIENT > "$CLIENT_CONF"
    [Interface]
    PrivateKey = $CLIENT_PRIV_KEY
    Address = $CLIENT_IP/32 # Client's specific IP within the VPN tunnel
    # Optional: Set a DNS server if you wish, e.g., your pfSense LAN IP (192.168.5.1)
    # DNS = 192.168.5.1

    [Peer]
    PublicKey = $SERVER_PUBLIC_KEY
    Endpoint = $SERVER_PUBLIC_IP:$SERVER_PORT
    AllowedIPs = 10.69.69.0/24, <YOUR_HOME_NETWORK_SUBNET> # e.g., 192.168.5.0/24
    PersistentKeepalive = 25
    EOCLIENT

      # -- ADD A [Peer] BLOCK TO THE SERVER CONFIG FRAGMENT --
      cat <<EOSERVER >> "$SERVER_CONFIG_FRAGMENT"
    # ---------------------------------------------------------------------
    # Client $i (IP: $CLIENT_IP)
    [Peer]
    PublicKey = $CLIENT_PUB_KEY
    AllowedIPs = $CLIENT_IP/32
    PersistentKeepalive = 25

    EOSERVER

      # -- PRINT THE QR CODE TO TERMINAL --
      echo ""
      echo "================================================================="
      echo "QR for Client $i -- $CLIENT_IP"
      echo "================================================================="
      qrencode -t ansiutf8 < "$CLIENT_CONF"
      echo ""

      ((i=i+1))
    done

    echo "==============================================================="
    echo "Generation complete!"
    echo "Server Peer fragment for clients is at: $SERVER_CONFIG_FRAGMENT"
    echo "Client configs are in: $OUTPUT_DIR/clientX.conf"
    echo "Above are the 10 QR codes in ANSI text form."
    echo "==============================================================="
    ```

    **Before saving, you MUST edit the following variables at the top of the script:**

    - `SERVER_PRIVATE_KEY`: Paste your VPS WireGuard server's private key here.
    - `SERVER_PUBLIC_IP`: Enter your VPS's public IP address or domain name.
    - `SERVER_WAN_IFACE`: Enter your VPS's public-facing network interface (e.g., `eth0`).
    - `AllowedIPs` in the client config section (`EOCLIENT` block): Replace `<YOUR_HOME_NETWORK_SUBNET>` with your actual home network's subnet (e.g., `192.168.5.0/24`). This allows the client to reach your home devices through the tunnel.

Save and exit the script.

### Step 3: Run the Script

Make the script executable and run it:

```bash
chmod +x generate_wireguard.sh
./generate_wireguard.sh
```

The script will:

- Generate 10 client `.conf` files in the `./output` directory.
- Print QR codes for each client directly in your terminal, which you can scan with the WireGuard app on your phone.
- Create a `server_peers_fragment.conf` file containing the `[Peer]` blocks for these 10 clients.

### Step 4: Add Client Peers to VPS `wg0.conf`

1.  View the content of the generated `server_peers_fragment.conf`:
    ```bash
    cat ./output/server_peers_fragment.conf
    ```
2.  Copy the entire output.
3.  Edit your main VPS WireGuard configuration file:
    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```
4.  Paste the copied `[Peer]` blocks to the end of your `wg0.conf` file.
5.  Save and exit.

### Step 5: Restart WireGuard on VPS

Apply the new configuration by restarting the WireGuard service on your VPS:

```bash
sudo systemctl restart wg-quick@wg0
sudo wg show
```

You should now see the newly added client peers when you run `sudo wg show`.

---

## Part 4: Configuring WireGuard Clients

1.  **Install WireGuard Client:** Download the WireGuard app for your mobile device (Android, iOS) or desktop (Windows, macOS, Linux).
2.  **Import Configuration:**
    - **QR Code (Easiest for Mobile):** In the WireGuard mobile app, select "Create from QR code" and scan the QR code printed by the script in your terminal.
    - **Import File:** On desktop, or if QR scanning isn't an option, you can `scp` the generated `.conf` files from your VPS to your local machine:
      ```bash
      scp root@<VPS_IP_ADDRESS>:/home/youruser/wireguard_client_gen/output/client1.conf /path/to/your/local/directory/
      ```
      Then, import `client1.conf` into your WireGuard client app.
3.  **Activate Tunnel:** In the WireGuard app, activate the newly imported tunnel.

### Verification

- **On your client device:** Once connected, try accessing devices on your home network (e.g., `192.168.5.100`).
- **On your VPS:** Run `sudo wg show` and observe the "latest handshake" timestamp for your client. This indicates a successful connection.
- **On your home router:** Check its WireGuard status to ensure it's connected to the VPS.

Done!
