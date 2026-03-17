# Pi-hole + DNScrypt-proxy + Traefik Docker Setup Guide

## Table of Contents
1. [Docker Networking Basics](#docker-networking-basics)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Setup Instructions](#setup-instructions)
5. [Configuration Details](#configuration-details)
6. [Verification and Testing](#verification-and-testing)
7. [Troubleshooting](#troubleshooting)

## Docker Networking Basics

### What is Docker Networking?

Docker networking allows containers to communicate with each other and with external systems. Think of it as creating virtual networks where your containers live.

### Key Concepts for Beginners

#### 1. **Network Types**

- **Bridge Network** (default): Containers on the same bridge network can talk to each other using container names as hostnames
- **Host Network**: Container shares the host's network stack (no isolation)
- **Custom Bridge Network**: User-defined networks with better isolation and automatic DNS resolution

#### 2. **Port Mapping**

When you see `-p 8080:80`, it means:
- `8080`: Port on your host machine (left side)
- `80`: Port inside the container (right side)
- Traffic to `localhost:8080` gets forwarded to port `80` inside the container

#### 3. **Container Name Resolution**

Containers on the same custom network can reach each other by container name:
- Container `pihole` can be reached at `http://pihole:80` by other containers
- No need to know IP addresses!

#### 4. **Network Isolation**

Containers on different networks cannot communicate unless explicitly connected to the same network.

## Architecture Overview

```
Internet → Traefik (Reverse Proxy) → Pi-hole (DNS + Ad Blocking) → DNScrypt-proxy (Encrypted DNS)
                ↓
           Your Devices
```

### How It Works

1. **DNScrypt-proxy**: Encrypts DNS queries and sends them to secure DNS servers (Cloudflare, Google, etc.)
2. **Pi-hole**: Acts as your network's DNS server, blocks ads/trackers, forwards queries to DNScrypt-proxy
3. **Traefik**: Provides secure HTTPS access to Pi-hole's web interface with automatic SSL certificates

### Why This Setup?

- **Privacy**: DNScrypt encrypts your DNS queries (prevents ISP snooping)
- **Ad-blocking**: Pi-hole blocks ads at the DNS level (works on all devices)
- **Security**: Traefik provides HTTPS access with automatic Let's Encrypt certificates
- **Convenience**: Easy web interface management

## Prerequisites

- Docker installed ([Install Docker](https://docs.docker.com/get-docker/))
- Docker Compose installed ([Install Docker Compose](https://docs.docker.com/compose/install/))
- Basic command line knowledge
- A domain name (optional, for Traefik SSL certificates)

## Setup Instructions

### Step 1: Create Project Directory

```bash
mkdir -p ~/pihole-stack
cd ~/pihole-stack
```

### Step 2: Create Docker Network

```bash
# Create a custom bridge network for container communication
docker network create pihole-net
```

### Step 3: Create Directory Structure

```bash
mkdir -p {pihole/etc-pihole,pihole/etc-dnsmasq.d,dnscrypt-proxy,traefik}
```

### Step 4: Create Docker Compose File

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # DNScrypt-proxy - Encrypted DNS
  dnscrypt-proxy:
    image: klutchell/dnscrypt-proxy:latest
    container_name: dnscrypt-proxy
    restart: unless-stopped
    networks:
      - pihole-net
    ports:
      - "5053:5053/udp"
      - "5053:5053/tcp"
    volumes:
      - ./dnscrypt-proxy:/config
    environment:
      - TZ=America/New_York  # Change to your timezone

  # Pi-hole - DNS and Ad Blocking
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    networks:
      - pihole-net
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"  # Web interface (will be proxied by Traefik)
    environment:
      TZ: 'America/New_York'  # Change to your timezone
      WEBPASSWORD: 'changeme123'  # Change this!
      PIHOLE_DNS_: '172.18.0.2#5053'  # DNScrypt-proxy address (adjust if needed)
      DNSSEC: 'false'  # DNScrypt handles this
      DNS_BOGUS_PRIV: 'true'
      DNS_FQDN_REQUIRED: 'true'
      REV_SERVER: 'false'
    volumes:
      - ./pihole/etc-pihole:/etc/pihole
      - ./pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    dns:
      - 127.0.0.1
      - 1.1.1.1  # Fallback DNS
    depends_on:
      - dnscrypt-proxy

  # Traefik - Reverse Proxy
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    networks:
      - pihole-net
    ports:
      - "80:80"
      - "443:443"
      - "8081:8080"  # Traefik dashboard
    command:
      # API and Dashboard
      - "--api.insecure=true"
      - "--api.dashboard=true"
      # Docker provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=pihole-net"
      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # HTTP to HTTPS redirect
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      # Let's Encrypt (commented out - enable when ready)
      # - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      # - "--certificatesresolvers.myresolver.acme.email=your-email@example.com"
      # - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/letsencrypt:/letsencrypt
    labels:
      - "traefik.enable=true"
      # Dashboard route
      - "traefik.http.routers.traefik.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.entrypoints=web"

networks:
  pihole-net:
    external: true
```

### Step 5: Configure DNScrypt-proxy

Create `dnscrypt-proxy/dnscrypt-proxy.toml`:

```toml
# DNScrypt-proxy configuration

# Listen addresses
listen_addresses = ['0.0.0.0:5053']

# Server selection
server_names = ['cloudflare', 'google', 'quad9-dnscrypt-ip4-filter-pri']

# Maximum number of active connections
max_clients = 250

# Use servers supporting DNS-over-HTTPS
doh_servers = true

# Require DNSSEC
require_dnssec = true

# Require NoLog servers
require_nolog = true

# Require NoFilter servers
require_nofilter = false

# Load balancing strategy
lb_strategy = 'p2'

# Cache settings
cache = true
cache_size = 4096
cache_min_ttl = 2400
cache_max_ttl = 86400
cache_neg_min_ttl = 60
cache_neg_max_ttl = 600

# Logging
log_level = 2
log_file = '/config/dnscrypt-proxy.log'

# Fallback resolver
fallback_resolvers = ['1.1.1.1:53', '8.8.8.8:53']
ignore_system_dns = true
netprobe_timeout = 60

# Block settings (optional)
[blocked_names]
  blocked_names_file = '/config/blocked-names.txt'

[blocked_ips]
  blocked_ips_file = '/config/blocked-ips.txt'
```

### Step 6: Add Traefik Labels to Pi-hole (Advanced)

If you want Traefik to proxy Pi-hole's web interface, add these labels to the `pihole` service in `docker-compose.yml`:

```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.pihole.rule=Host(`pihole.localhost`)"  # Change to your domain
      - "traefik.http.routers.pihole.entrypoints=websecure"
      - "traefik.http.routers.pihole.tls=true"
      # - "traefik.http.routers.pihole.tls.certresolver=myresolver"  # Uncomment for Let's Encrypt
      - "traefik.http.services.pihole.loadbalancer.server.port=80"
```

### Step 7: Start the Stack

```bash
# Start all containers
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

## Configuration Details

### Finding DNScrypt-proxy Container IP

The `PIHOLE_DNS_` environment variable needs DNScrypt-proxy's IP address. Here's how to find it:

```bash
# Get DNScrypt-proxy IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dnscrypt-proxy
```

Update the Pi-hole `PIHOLE_DNS_` variable with this IP:
```yaml
PIHOLE_DNS_: '<dnscrypt-ip>#5053'
```

**Better Approach**: Use Docker's DNS resolution:
```yaml
PIHOLE_DNS_: 'dnscrypt-proxy#5053'
```

### Pi-hole DNS Configuration

After starting, configure Pi-hole to use DNScrypt-proxy:

1. Access Pi-hole web interface: `http://localhost:8080/admin`
2. Login with the password from `WEBPASSWORD`
3. Go to **Settings** → **DNS**
4. Uncheck all **Upstream DNS Servers**
5. Add custom DNS: `dnscrypt-proxy#5053` or use the container IP
6. Save settings

### Traefik Configuration

#### Local Testing (No Domain)

For local testing without SSL:
- Pi-hole: `http://localhost:8080`
- Traefik Dashboard: `http://localhost:8081`

#### Production (With Domain)

1. Point your domain to your server's IP
2. Uncomment Let's Encrypt lines in `docker-compose.yml`
3. Change `pihole.localhost` to `pihole.yourdomain.com`
4. Update email in ACME configuration
5. Restart Traefik: `docker-compose restart traefik`

### Network Configuration on Your Devices

To use Pi-hole network-wide:

#### Option 1: Configure Individual Devices
Set DNS server to your Pi-hole server's IP address (e.g., `192.168.1.100`)

#### Option 2: Configure Your Router (Recommended)
Set DHCP DNS server to your Pi-hole server's IP address. All devices will automatically use Pi-hole.

## Verification and Testing

### Check Container Status

```bash
# All containers should be "Up"
docker-compose ps

# Check logs for errors
docker-compose logs dnscrypt-proxy | tail -20
docker-compose logs pihole | tail -20
docker-compose logs traefik | tail -20
```

### Test DNScrypt-proxy

```bash
# Test DNS resolution through DNScrypt-proxy
docker exec dnscrypt-proxy sh -c "nslookup google.com 127.0.0.1"
```

### Test Pi-hole

```bash
# Query Pi-hole directly
nslookup google.com localhost

# Test ad blocking (should be blocked)
nslookup doubleclick.net localhost
```

### Test Network Connectivity Between Containers

```bash
# From Pi-hole, ping DNScrypt-proxy
docker exec pihole ping -c 3 dnscrypt-proxy

# Check if Pi-hole can resolve through DNScrypt-proxy
docker exec pihole nslookup google.com dnscrypt-proxy
```

### Access Web Interfaces

- **Pi-hole**: `http://localhost:8080/admin`
- **Traefik Dashboard**: `http://localhost:8081`

## Troubleshooting

### Common Issues

#### 1. Pi-hole Can't Reach DNScrypt-proxy

**Symptoms**: DNS queries fail in Pi-hole logs

**Solution**:
```bash
# Check if containers are on the same network
docker network inspect pihole-net

# Ensure both containers are listed
# Get DNScrypt-proxy IP and update Pi-hole config
docker inspect dnscrypt-proxy | grep IPAddress
```

#### 2. Port Conflicts

**Symptoms**: "port is already allocated" error

**Solution**:
```bash
# Check what's using port 53
sudo lsof -i :53

# On Linux, stop systemd-resolved if needed
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

#### 3. DNS Not Working on Host

**Symptoms**: Host machine can't resolve DNS

**Solution**:
- This is normal - containers have their own network
- Change your host's DNS to `127.0.0.1` if Pi-hole runs on the host
- Or access Pi-hole via its container IP or bridge network IP

#### 4. Traefik Not Proxying Pi-hole

**Symptoms**: Can't access `pihole.localhost`

**Solution**:
```bash
# Check Traefik logs
docker-compose logs traefik

# Ensure labels are correct on Pi-hole container
docker inspect pihole | grep -A 20 Labels

# Verify Traefik can see Pi-hole
docker exec traefik wget -O- http://pihole
```

#### 5. Let's Encrypt Certificate Fails

**Symptoms**: No SSL certificate, browser shows insecure

**Solution**:
- Ensure your domain points to your server
- Check port 80 and 443 are accessible from internet
- Review Traefik logs: `docker-compose logs traefik | grep acme`
- Verify email is set correctly in ACME configuration

### Useful Commands

```bash
# Restart a specific service
docker-compose restart pihole

# Stop all services
docker-compose down

# Stop and remove volumes (⚠️ deletes data)
docker-compose down -v

# View real-time logs
docker-compose logs -f pihole

# Execute command in container
docker exec -it pihole bash

# Check DNS forwarding in Pi-hole
docker exec pihole cat /etc/pihole/setupVars.conf | grep PIHOLE_DNS
```

## Docker Networking Deep Dive

### How Containers Communicate in This Setup

1. **Custom Bridge Network** (`pihole-net`):
   - All containers join this network
   - Docker provides DNS resolution (container name → IP)
   - Isolated from other Docker networks

2. **DNS Resolution**:
   ```
   pihole container → dnscrypt-proxy (by name) → resolved to IP automatically
   ```

3. **Port Publishing**:
   - `53:53` → Exposes Pi-hole DNS to host
   - `8080:80` → Exposes Pi-hole web UI to host
   - Containers can reach each other without port publishing

### Network Flow Example

When you query `google.com`:

```
Your Device (192.168.1.50)
    ↓ DNS query on port 53
Pi-hole Container (172.18.0.3:53)
    ↓ Checks block lists → Not blocked
    ↓ Forwards to upstream DNS
DNScrypt-proxy Container (172.18.0.2:5053)
    ↓ Encrypts query
    ↓ Sends to Cloudflare/Google
Cloudflare DNS (1.1.1.1)
    ↓ Returns answer
DNScrypt-proxy → Decrypts
    ↓
Pi-hole → Caches
    ↓
Your Device → Gets response
```

## Advanced Topics

### Using Custom Blocklists

Add blocklists in Pi-hole web interface:
1. Go to **Group Management** → **Adlists**
2. Add URLs of blocklists (e.g., from [firebog.net](https://firebog.net))
3. Run **Tools** → **Update Gravity**

### Monitoring and Logs

```bash
# Pi-hole query log
docker exec pihole tail -f /var/log/pihole.log

# DNScrypt-proxy log
docker exec dnscrypt-proxy tail -f /config/dnscrypt-proxy.log

# Traefik access log
docker-compose logs -f traefik
```

### Backup Configuration

```bash
# Backup Pi-hole configuration
tar -czf pihole-backup-$(date +%Y%m%d).tar.gz pihole/

# Backup entire stack
tar -czf pihole-stack-backup-$(date +%Y%m%d).tar.gz .
```

### Performance Tuning

In `dnscrypt-proxy.toml`:
```toml
# Increase cache size for better performance
cache_size = 8192

# Adjust timeout
timeout = 5000

# Use more servers for redundancy
server_names = ['cloudflare', 'google', 'quad9-dnscrypt-ip4-filter-pri', 'cloudflare-ipv6']
```

## Resources

- [Pi-hole Docker Documentation](https://github.com/pi-hole/docker-pi-hole)
- [DNScrypt-proxy Docker Documentation](https://github.com/klutchell/dnscrypt-proxy-docker)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Docker Networking Overview](https://docs.docker.com/network/)
- [Docker Compose Networking](https://docs.docker.com/compose/networking/)

## Conclusion

This setup provides:
- ✅ Network-wide ad blocking (Pi-hole)
- ✅ Encrypted DNS queries (DNScrypt-proxy)
- ✅ Secure web interface access (Traefik)
- ✅ Containerized, portable, and easy to maintain

The key to understanding this stack is Docker networking - containers communicate via the custom bridge network, using container names as hostnames. Traefik acts as the front door, Pi-hole filters DNS queries, and DNScrypt-proxy encrypts them for privacy.

Happy ad-free, privacy-respecting browsing!
