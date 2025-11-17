# Sentinel-X Server with Maximum Security via Tor Hidden Services v3
(outdated: new Version 6.4.5 with "no Clearnet Option / SSH over Tor V3" on the Way)

<p align="center">
  <img src="bg.png" alt="Vorschau" width="1920">
</p>


## Configuration Variants Overview

This guide offers two variants:

**Variant A: Clearnet-Compatible** 
Server also operates other services (Matrix, Nginx, etc.) that must be accessible from the internet. SimpleX runs only via Tor.

**Variant B: Tor-Only** 
Server is completely isolated, NOTHING is accessible from clearnet. Maximum security.

---

## 1. System Preparation (Debian 12)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y \
    curl wget gnupg2 lsb-release \
    iptables iptables-persistent \
    python3-venv python3-pip python3-stem \
    jq bc mailutils \
    fail2ban git netcat-openbsd lsof xxd openssl

# Create SimpleX user (if not exists)
id simplex >/dev/null 2>&1 || sudo useradd -r -s /bin/false -m -d /var/opt/simplex simplex

# Create directories
sudo mkdir -p /etc/opt/simplex /var/opt/simplex/{smp,xftp-files,logs}
sudo chown -R simplex:simplex /var/opt/simplex
sudo chmod 750 /var/opt/simplex
```

## 2. Tor Installation (Latest Version)

```bash
# Add Tor Project GPG key
curl -fsSL https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | \
    sudo gpg --dearmor -o /usr/share/keyrings/tor-archive-keyring.gpg

# Add Tor repository
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] https://deb.torproject.org/torproject.org bookworm main" | \
    sudo tee /etc/apt/sources.list.d/tor.list

# Install Tor
sudo apt update
sudo apt install -y tor deb.torproject.org-keyring
```

## 3. SimpleX Server Installation

```bash
# Download SimpleX v6.4.1
SMP_VERSION="v6.4.1"
GITHUB_BASE="https://github.com/simplex-chat/simplexmq/releases/download"

# Download SMP server
sudo curl -fL "${GITHUB_BASE}/${SMP_VERSION}/smp-server-ubuntu-22_04-x86-64" \
    -o /usr/local/bin/smp-server

# Make executable
sudo chmod 0755 /usr/local/bin/smp-server
sudo chown root:root /usr/local/bin/smp-server
```

## 4. Configure Tor Hidden Service

```bash
# Customize Tor configuration
sudo tee /etc/tor/torrc > /dev/null << 'EOF'
# === Basics ===
DataDirectory /var/lib/tor
PidFile /run/tor/tor.pid
RunAsDaemon 1
User debian-tor

# === Logging ===
Log notice file /var/log/tor/notices.log

# === Control Port ===
ControlPort 127.0.0.1:9051
CookieAuthentication 1

# === SOCKS Port ===
SocksPort 127.0.0.1:9050
SocksPolicy accept 127.0.0.1
SocksPolicy reject *

# === IMPORTANT: Client Auth Directory for outgoing connections ===
ClientOnionAuthDir /var/lib/tor/client_auth

# === Hidden Service for SMP ===
HiddenServiceDir /var/lib/tor/simplex-smp
HiddenServiceVersion 3
HiddenServicePort 5223 127.0.0.1:5223
EOF

# Create client auth directory
sudo mkdir -p /var/lib/tor/client_auth
sudo chown debian-tor:debian-tor /var/lib/tor/client_auth
sudo chmod 700 /var/lib/tor/client_auth

# Restart Tor
sudo systemctl restart tor@default

# Display onion address (after restart)
sleep 5
echo "SMP Onion Address:"
sudo cat /var/lib/tor/simplex-smp/hostname
```

## 5. Client Authentication Setup

```bash
# Create working directory
mkdir -p ~/tor-auth/smp
cd ~/tor-auth/smp

# Generate keys for Windows client
openssl genpkey -algorithm X25519 -out smp_windows.pem
openssl pkey -outform DER -in smp_windows.pem | tail -c 32 | base32 | tr -d '=' > smp_windows_private.key
openssl pkey -outform DER -in smp_windows.pem -pubout | tail -c 32 | base32 | tr -d '=' > smp_windows_public.key

# Generate keys for Android client
openssl genpkey -algorithm X25519 -out smp_android.pem
openssl pkey -outform DER -in smp_android.pem | tail -c 32 | base32 | tr -d '=' > smp_android_private.key
openssl pkey -outform DER -in smp_android.pem -pubout | tail -c 32 | base32 | tr -d '=' > smp_android_public.key

# Add public keys to Tor
sudo mkdir -p /var/lib/tor/simplex-smp/authorized_clients
echo "descriptor:x25519:$(cat smp_windows_public.key)" | sudo tee /var/lib/tor/simplex-smp/authorized_clients/windows.auth
echo "descriptor:x25519:$(cat smp_android_public.key)" | sudo tee /var/lib/tor/simplex-smp/authorized_clients/android.auth

# Set permissions
sudo chown -R debian-tor:debian-tor /var/lib/tor/simplex-smp/authorized_clients/
sudo chmod 600 /var/lib/tor/simplex-smp/authorized_clients/*.auth

# Reload Tor
sudo systemctl reload tor@default

# Display keys
echo "=== CLIENT KEYS ==="
echo "Windows Private Key: $(cat smp_windows_private.key)"
echo "Android Private Key: $(cat smp_android_private.key)"
```

## 6. SimpleX Server Configuration

```bash
# Initialize SMP server
sudo -u simplex /usr/local/bin/smp-server init

# Edit configuration
sudo nano /etc/opt/simplex/smp-server.ini
```

Important changes in configuration:

1. **host: parameter** (ONLY for display, NOT for binding!):
```ini
[TRANSPORT]
host: 127.0.0.1
port: 5223
```

2. **Configure PROXY section** (for outgoing Tor connections):
```ini
[PROXY]
socks_proxy: 127.0.0.1:9050
socks_mode: onion
```

3. **Password for new queues** (should be automatically set):
```ini
[TRANSPORT]
create_password: [automatically generated password]
```

## 7. Firewall Configuration

### Variant A: Clearnet-Compatible (other services remain accessible)

```bash
# Install iptables-persistent (if not already installed)
sudo apt install -y iptables-persistent

# Allow port 5223 only for localhost, block from outside
sudo iptables -I INPUT -i lo -p tcp --dport 5223 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5223 -j DROP

# For IPv6 as well
sudo ip6tables -I INPUT -i lo -p tcp --dport 5223 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5223 -j DROP

# Save rules
sudo netfilter-persistent save

# Test: Port 5223 should NOT be accessible from outside
nc -zv $(hostname -I | awk '{print $1}') 5223  # Should fail
nc -zv 127.0.0.1 5223                          # Should work
```

### Variant B: Tor-Only (complete isolation)

```bash
# Block all incoming connections, only allow localhost
sudo iptables -P INPUT DROP
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Only SSH temporarily for setup (remove later!)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Outgoing only via Tor
sudo iptables -A OUTPUT -m owner --uid-owner debian-tor -j ACCEPT
sudo iptables -A OUTPUT -d 127.0.0.0/8 -j ACCEPT
sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -j DROP

# Save
sudo netfilter-persistent save
```

## 8. Create Systemd Service

```bash
sudo tee /etc/systemd/system/smp-server.service > /dev/null << 'EOF'
[Unit]
Description=SimpleX SMP Server
After=network.target tor@default.service
Requires=tor@default.service

[Service]
Type=simple
User=simplex
Group=simplex
WorkingDirectory=/var/opt/simplex
ExecStart=/usr/local/bin/smp-server start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable --now smp-server

# Check status
sudo systemctl status smp-server
```

## 9. Server-to-Server Authentication (for multiple own servers)

**IMPORTANT**: If you operate multiple own SMP servers, they MUST be able to authenticate each other!

### On Server 1:
```bash
cd ~/tor-auth/smp

# Generate keys for server-to-server
openssl genpkey -algorithm X25519 -out server_relay.pem
openssl pkey -outform DER -in server_relay.pem | tail -c 32 | base32 | tr -d '=' > server_relay_private.key
openssl pkey -outform DER -in server_relay.pem -pubout | tail -c 32 | base32 | tr -d '=' > server_relay_public.key

echo "Public Key for Server 2: $(cat server_relay_public.key)"
echo "Private Key for this server: $(cat server_relay_private.key)"
```

### On Server 2:
```bash
# Same commands as Server 1
# Then exchange keys
```

### Install keys:

On Server 1:
```bash
# Add public key from Server 2
echo "descriptor:x25519:[PUBLIC_KEY_FROM_SERVER2]" | \
    sudo tee /var/lib/tor/simplex-smp/authorized_clients/server2.auth
sudo chmod 600 /var/lib/tor/simplex-smp/authorized_clients/server2.auth

# Private key for outgoing connections to Server 2
echo "[ONION_FROM_SERVER2]:descriptor:x25519:[PRIVATE_KEY_FROM_SERVER1]" | \
    sudo tee /var/lib/tor/client_auth/server2.auth_private
sudo chmod 600 /var/lib/tor/client_auth/server2.auth_private
```

On Server 2 vice versa, then reload both Tor services:
```bash
sudo systemctl reload tor@default
```

## 10. Vanguards Installation (Optional but recommended)

```bash
cd /tmp
git clone https://github.com/mikeperry-tor/vanguards
cd vanguards
sudo python3 -m venv /opt/vanguards-env
sudo /opt/vanguards-env/bin/pip install stem pyyaml
sudo /opt/vanguards-env/bin/python setup.py install

# Create service
sudo tee /etc/systemd/system/vanguards.service > /dev/null << 'EOF'
[Unit]
Description=Vanguards - Tor Guard Discovery Protection
After=tor@default.service
Requires=tor@default.service

[Service]
Type=simple
User=debian-tor
Group=debian-tor
ExecStart=/opt/vanguards-env/bin/python /opt/vanguards-env/bin/vanguards
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vanguards
```

## 11. Generate SimpleX Server Link

```bash
# Get fingerprint
FINGERPRINT=$(sudo journalctl -u smp-server | grep "Fingerprint" | tail -1 | cut -d' ' -f5)

# Get password
PASSWORD=$(sudo grep "create_password:" /etc/opt/simplex/smp-server.ini | cut -d' ' -f2)

# Onion address
ONION=$(sudo cat /var/lib/tor/simplex-smp/hostname)

echo "=== SIMPLEX SERVER LINK ==="
echo "smp://${FINGERPRINT}:${PASSWORD}@${ONION}"
```

## 12. Windows Client Configuration

### Install and configure Tor:
```powershell
# Extract to C:\tor\
# Create torrc
@"
SocksPort 9050
ClientOnionAuthDir C:\tor\tor
"@ | Out-File -FilePath "C:\tor\tor\torrc" -Encoding ASCII

# Create auth file (onion address without .onion!)
$onion = "uru2k3xdu4tovq7qwtrgl7zhexobeo4wsjrind33fsigwkonj5wrdhqd"
$privateKey = "DC3PX7PUX3XRB5BVVQRXNFOF6XATSSLP4MCSPJ3FPABAQWASGNEA"

"${onion}:descriptor:x25519:${privateKey}" | 
    Out-File -FilePath "C:\tor\tor\${onion}.auth_private" -Encoding ASCII -NoNewline

# Start Tor
.\tor.exe -f "C:\tor\tor\torrc"
```

## Troubleshooting

### Problem: "Private Routing error"
**Cause**: Servers cannot reach each other
**Solution**: Perform Step 9 (Server-to-Server Auth)

### Problem: Server listens on all interfaces
**Cause**: SimpleX has no bind option
**Solution**: Firewall rules (Step 7) MUST be active!

### Problem: PCENetworkError in logs
**Cause**: ClientOnionAuthDir missing in torrc
**Solution**: Add line to torrc and restart Tor

## Installation Verification

```bash
# 1. Service running?
sudo systemctl status smp-server

# 2. Port only accessible on localhost?
nc -zv 127.0.0.1 5223          # Should work
nc -zv $(hostname -I | awk '{print $1}') 5223  # Should NOT work

# 3. Tor connection working?
curl --socks5 127.0.0.1:9050 https://check.torproject.org/api/ip

# 4. Display server link
sudo journalctl -u smp-server | grep "Server address"
```

## Important Security Notes

- **SimpleX servers ALWAYS bind to all interfaces** - Firewall is CRITICAL!
- Never store or share private keys in plain text
- With multiple servers, server-to-server keys MUST be configured
- ClientOnionAuthDir in torrc is MANDATORY for outgoing connections
- Regular backups of `/var/lib/tor/` directories

---

*Version 6.0 - With all fixes for multi-server setup and clearnet compatibility*
*Last Updated: August 2025 | SimpleX v6.4.1 | Debian 12*
