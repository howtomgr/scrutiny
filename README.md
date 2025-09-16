# Scrutiny Installation Guide

Scrutiny is a free and open-source S.M.A.R.T. disk monitoring tool that provides web-based dashboards, alerting, and historical data analysis for hard drive health. It serves as a FOSS alternative to proprietary disk monitoring solutions like CrystalDiskInfo, Hard Disk Sentinel, or enterprise storage monitoring tools, offering comprehensive disk health monitoring with beautiful visualizations and proactive failure detection.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

### Hardware Requirements
- **CPU**: 1+ cores
- **RAM**: 512MB minimum (1GB+ recommended)
- **Storage**: 100MB for application, additional for metrics storage
- **Disks**: SATA, NVMe, or SAS drives with S.M.A.R.T. support

### Software Requirements
- **smartctl**: smartmontools package for S.M.A.R.T. data collection
- **Docker**: Optional for containerized deployment
- **Database**: SQLite (built-in) or InfluxDB for metrics storage

### Network Requirements
- **Ports**: 
  - 8080: Web interface (default)
  - 8086: InfluxDB (if used)

## 2. Supported Operating Systems

Scrutiny officially supports:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04 LTS / 22.04 LTS / 24.04 LTS
- Arch Linux
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- Fedora 38+
- macOS 12+ (limited S.M.A.R.T. support)
- Windows 10/11 (via WSL2 or native)

## 3. Installation

### Method 1: Docker Compose (Recommended)

#### All Linux Distributions

```bash
# Install Docker and Docker Compose
# For RHEL/CentOS/Rocky/AlmaLinux:
sudo dnf install -y docker docker-compose
sudo systemctl enable --now docker

# For Debian/Ubuntu:
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker

# For Arch Linux:
sudo pacman -S docker docker-compose
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Create Scrutiny directory
mkdir -p ~/scrutiny
cd ~/scrutiny

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  scrutiny:
    image: ghcr.io/analogj/scrutiny:master-omnibus
    container_name: scrutiny
    cap_add:
      - SYS_RAWIO
      - SYS_ADMIN
    ports:
      - "8080:8080"
    volumes:
      - /run/udev:/run/udev:ro
      - ./config:/opt/scrutiny/config
      - ./influxdb:/opt/scrutiny/influxdb
    devices:
      - /dev/sda
      - /dev/sdb
      - /dev/nvme0n1
    environment:
      - SCRUTINY_WEB_LISTEN_PORT=8080
      - SCRUTINY_WEB_LISTEN_HOST=0.0.0.0
    restart: unless-stopped
EOF

# Start Scrutiny
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs scrutiny
```

### Method 2: Native Installation

#### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Install dependencies
sudo dnf install -y smartmontools wget curl

# Download Scrutiny binary
SCRUTINY_VERSION="0.7.3"  # Check latest version
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-web-linux-amd64"
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-collector-metrics-linux-amd64"

# Install binaries
sudo mv scrutiny-web-linux-amd64 /usr/local/bin/scrutiny-web
sudo mv scrutiny-collector-metrics-linux-amd64 /usr/local/bin/scrutiny-collector-metrics
sudo chmod +x /usr/local/bin/scrutiny-*

# Create scrutiny user
sudo useradd -r -s /bin/false -d /opt/scrutiny scrutiny

# Create directories
sudo mkdir -p /opt/scrutiny/{config,data}
sudo chown -R scrutiny:scrutiny /opt/scrutiny

# Create configuration
sudo tee /opt/scrutiny/config/scrutiny.yaml > /dev/null << 'EOF'
version: 1

web:
  listen:
    port: 8080
    host: 0.0.0.0
  database:
    location: /opt/scrutiny/data/scrutiny.db
  src:
    frontend:
      path: ./dist

log:
  level: INFO
EOF

# Create systemd service
sudo tee /etc/systemd/system/scrutiny-web.service > /dev/null << 'EOF'
[Unit]
Description=Scrutiny Web Dashboard
After=network.target

[Service]
Type=simple
User=scrutiny
Group=scrutiny
ExecStart=/usr/local/bin/scrutiny-web start --config /opt/scrutiny/config/scrutiny.yaml
WorkingDirectory=/opt/scrutiny
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable --now scrutiny-web
```

#### Debian/Ubuntu

```bash
# Update system and install dependencies
sudo apt update
sudo apt install -y smartmontools wget curl

# Download and install Scrutiny
SCRUTINY_VERSION="0.7.3"
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-web-linux-amd64"
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-collector-metrics-linux-amd64"

sudo mv scrutiny-web-linux-amd64 /usr/local/bin/scrutiny-web
sudo mv scrutiny-collector-metrics-linux-amd64 /usr/local/bin/scrutiny-collector-metrics
sudo chmod +x /usr/local/bin/scrutiny-*

# Create user and directories
sudo useradd -r -s /bin/false -d /opt/scrutiny scrutiny
sudo mkdir -p /opt/scrutiny/{config,data}
sudo chown -R scrutiny:scrutiny /opt/scrutiny

# Install configuration (same as RHEL above)
# Install systemd service (same as RHEL above)

sudo systemctl daemon-reload
sudo systemctl enable --now scrutiny-web
```

#### Arch Linux

```bash
# Install from AUR
yay -S scrutiny-bin

# Or install dependencies and manual setup
sudo pacman -S smartmontools

# Follow manual installation steps
# (Same binary download and setup as above)
```

#### Alpine Linux

```bash
# Install dependencies
apk add --no-cache smartmontools wget

# Download Scrutiny binaries
SCRUTINY_VERSION="0.7.3"
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-web-linux-amd64"
wget "https://github.com/AnalogJ/scrutiny/releases/download/v${SCRUTINY_VERSION}/scrutiny-collector-metrics-linux-amd64"

mv scrutiny-web-linux-amd64 /usr/local/bin/scrutiny-web
mv scrutiny-collector-metrics-linux-amd64 /usr/local/bin/scrutiny-collector-metrics
chmod +x /usr/local/bin/scrutiny-*

# Create OpenRC service
tee /etc/init.d/scrutiny-web > /dev/null << 'EOF'
#!/sbin/openrc-run

description="Scrutiny Web Dashboard"
command="/usr/local/bin/scrutiny-web"
command_args="start --config /opt/scrutiny/config/scrutiny.yaml"
command_user="scrutiny"
command_group="scrutiny"
pidfile="/run/scrutiny-web.pid"
command_background="yes"

depend() {
    need net
    after firewall
}

start_pre() {
    checkpath --directory --owner scrutiny:scrutiny --mode 0755 /opt/scrutiny/data
}
EOF

chmod +x /etc/init.d/scrutiny-web
rc-update add scrutiny-web default
rc-service scrutiny-web start
```

### Method 3: Metrics Collector Setup

```bash
# Create collector configuration
sudo tee /opt/scrutiny/config/collector.yaml > /dev/null << 'EOF'
version: 1

host:
  id: ""

api:
  endpoint: "http://localhost:8080"

log:
  level: INFO

commands:
  metrics_scan_interval: 15m
EOF

# Create collector systemd service
sudo tee /etc/systemd/system/scrutiny-collector.service > /dev/null << 'EOF'
[Unit]
Description=Scrutiny Collector
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/scrutiny-collector-metrics run --config /opt/scrutiny/config/collector.yaml
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now scrutiny-collector
```

## 4. Configuration

### Web Dashboard Configuration

Edit `/opt/scrutiny/config/scrutiny.yaml`:
```yaml
version: 1

web:
  listen:
    port: 8080
    host: 0.0.0.0
  
  database:
    location: /opt/scrutiny/data/scrutiny.db
  
  influxdb:
    host: localhost
    port: 8086
    token: ""
    org: ""
    bucket: "metrics"
    retention_policy: ""

notify:
  urls:
    - "discord://webhook_id/webhook_token"
    - "slack://webhook_url"
    - "smtp://username:password@host:port/?to=user@example.com"

log:
  level: INFO
  file: "/opt/scrutiny/data/web.log"
```

### Collector Configuration

Edit `/opt/scrutiny/config/collector.yaml`:
```yaml
version: 1

host:
  id: ""

api:
  endpoint: "http://localhost:8080"

commands:
  metrics_scan_interval: 15m
  
devices:
  include:
    - /dev/sd*
    - /dev/nvme*
  exclude:
    - /dev/loop*
    - /dev/sr*

log:
  level: INFO
  file: "/opt/scrutiny/data/collector.log"
```

### Notification Setup

```yaml
# Discord Webhook
notify:
  urls:
    - "discord://webhook_id/webhook_token?username=Scrutiny"

# Slack Webhook
notify:
  urls:
    - "slack://webhook_url"

# Email SMTP
notify:
  urls:
    - "smtp://user:pass@smtp.gmail.com:587/?to=admin@example.com&subject=Scrutiny%20Alert"

# Multiple notifications
notify:
  urls:
    - "discord://webhook_id/webhook_token"
    - "smtp://user:pass@smtp.gmail.com:587/?to=admin@example.com"
```

## 5. Service Management

### systemd Management

```bash
# Web dashboard
sudo systemctl start scrutiny-web
sudo systemctl stop scrutiny-web
sudo systemctl restart scrutiny-web
sudo systemctl status scrutiny-web
sudo systemctl enable scrutiny-web

# Collector
sudo systemctl start scrutiny-collector
sudo systemctl stop scrutiny-collector
sudo systemctl restart scrutiny-collector
sudo systemctl status scrutiny-collector

# View logs
sudo journalctl -u scrutiny-web -f
sudo journalctl -u scrutiny-collector -f
```

### Docker Management

```bash
# Start/stop services
docker-compose up -d
docker-compose down
docker-compose restart

# View logs
docker-compose logs -f scrutiny

# Update to latest version
docker-compose pull
docker-compose up -d
```

### Manual Operations

```bash
# Run one-time disk scan
sudo /usr/local/bin/scrutiny-collector-metrics run --config /opt/scrutiny/config/collector.yaml

# Test SMART data collection
sudo smartctl -a /dev/sda
sudo smartctl -H /dev/nvme0n1

# Force metrics collection
curl -X POST http://localhost:8080/api/health/notify
```

## 6. Troubleshooting

### Common Issues

1. **No disks detected**:
```bash
# Check if smartctl can detect disks
sudo smartctl --scan

# Verify disk permissions
ls -la /dev/sd* /dev/nvme*

# Check if disks support SMART
sudo smartctl -i /dev/sda
sudo smartctl -c /dev/sda
```

2. **Permission errors**:
```bash
# Ensure scrutiny user has access to disks
sudo usermod -a -G disk scrutiny

# For Docker, check device mapping
docker-compose exec scrutiny smartctl --scan

# Check udev rules
ls -la /run/udev
```

3. **Web interface not accessible**:
```bash
# Check if service is running
sudo systemctl status scrutiny-web

# Verify port binding
sudo netstat -tlnp | grep 8080

# Check firewall
sudo firewall-cmd --list-ports
sudo ufw status
```

4. **Database issues**:
```bash
# Check database permissions
ls -la /opt/scrutiny/data/scrutiny.db

# Verify SQLite installation
sqlite3 --version

# Check database integrity
sqlite3 /opt/scrutiny/data/scrutiny.db "PRAGMA integrity_check;"
```

### Debug Mode

```bash
# Enable debug logging in configuration
log:
  level: DEBUG

# Run collector manually with debug
sudo /usr/local/bin/scrutiny-collector-metrics run \
    --config /opt/scrutiny/config/collector.yaml \
    --log-level DEBUG

# Check SMART test results
sudo smartctl -l selftest /dev/sda
```

## 7. Security Considerations

### Access Control

```bash
# Bind to localhost only (behind reverse proxy)
web:
  listen:
    port: 8080
    host: 127.0.0.1

# Configure reverse proxy with authentication
# nginx example:
location /scrutiny {
    auth_basic "Scrutiny Access";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:8080;
}
```

### Firewall Configuration

```bash
# firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# UFW (Ubuntu/Debian)
sudo ufw allow 8080/tcp
sudo ufw enable

# iptables
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

### File Permissions

```bash
# Secure configuration files
sudo chmod 640 /opt/scrutiny/config/*.yaml
sudo chown scrutiny:scrutiny /opt/scrutiny/config/*.yaml

# Secure data directory
sudo chmod 750 /opt/scrutiny/data
sudo chown -R scrutiny:scrutiny /opt/scrutiny/data
```

### SSL/TLS Setup

```nginx
# nginx reverse proxy with SSL
server {
    listen 443 ssl;
    server_name scrutiny.example.com;
    
    ssl_certificate /etc/ssl/certs/scrutiny.crt;
    ssl_certificate_key /etc/ssl/private/scrutiny.key;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 8. Performance Tuning

### Collector Optimization

```yaml
# Adjust scan intervals based on needs
commands:
  metrics_scan_interval: 30m  # Longer intervals for less load
  
# Exclude unnecessary devices
devices:
  exclude:
    - /dev/loop*
    - /dev/sr*
    - /dev/ram*
```

### Database Optimization

```bash
# Optimize SQLite database
sqlite3 /opt/scrutiny/data/scrutiny.db "VACUUM;"
sqlite3 /opt/scrutiny/data/scrutiny.db "ANALYZE;"

# Set SQLite performance pragmas
sqlite3 /opt/scrutiny/data/scrutiny.db "PRAGMA journal_mode = WAL;"
sqlite3 /opt/scrutiny/data/scrutiny.db "PRAGMA synchronous = NORMAL;"
```

### System Resource Optimization

```bash
# Limit memory usage for Docker
docker-compose.yml:
services:
  scrutiny:
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

# CPU limits
    deploy:
      resources:
        limits:
          cpus: '1.0'
```

## 9. Backup and Restore

### Database Backup

```bash
#!/bin/bash
# backup-scrutiny-db.sh

BACKUP_DIR="/var/backups/scrutiny"
DATE=$(date +%Y%m%d_%H%M%S)
DB_PATH="/opt/scrutiny/data/scrutiny.db"

mkdir -p $BACKUP_DIR

# Stop service for consistent backup
sudo systemctl stop scrutiny-web

# Backup database
sudo cp $DB_PATH $BACKUP_DIR/scrutiny_db_$DATE.db

# Compress backup
gzip $BACKUP_DIR/scrutiny_db_$DATE.db

# Start service
sudo systemctl start scrutiny-web

echo "Database backup completed: $BACKUP_DIR/scrutiny_db_$DATE.db.gz"
```

### Configuration Backup

```bash
#!/bin/bash
# backup-scrutiny-config.sh

BACKUP_DIR="/var/backups/scrutiny"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup configuration
tar -czf $BACKUP_DIR/scrutiny_config_$DATE.tar.gz \
    /opt/scrutiny/config/ \
    /etc/systemd/system/scrutiny*.service

echo "Configuration backup completed"
```

### Automated Backup

```bash
# Add to crontab
sudo crontab -e

# Daily database backup at 2 AM
0 2 * * * /opt/scrutiny/scripts/backup-scrutiny-db.sh

# Weekly configuration backup
0 3 * * 0 /opt/scrutiny/scripts/backup-scrutiny-config.sh
```

### Restore Procedures

```bash
# Restore database
sudo systemctl stop scrutiny-web
sudo cp scrutiny_db_backup.db /opt/scrutiny/data/scrutiny.db
sudo chown scrutiny:scrutiny /opt/scrutiny/data/scrutiny.db
sudo systemctl start scrutiny-web

# Restore configuration
sudo tar -xzf scrutiny_config_backup.tar.gz -C /
sudo systemctl daemon-reload
sudo systemctl restart scrutiny-web scrutiny-collector
```

## 10. System Requirements

### Minimum Requirements
- **CPU**: 1 core
- **RAM**: 512MB
- **Storage**: 100MB + metrics storage
- **Disks**: SMART-enabled drives

### Recommended Requirements
- **CPU**: 2+ cores
- **RAM**: 1GB+
- **Storage**: 1GB+ SSD for database
- **Network**: Fast local network for multi-host setup

### Scaling Guidelines
- **Per disk**: ~1MB metrics data per day
- **Retention**: Plan storage for historical data
- **Collection frequency**: Balance between monitoring granularity and system load

## 11. Support

### Official Resources
- **GitHub**: https://github.com/AnalogJ/scrutiny
- **Documentation**: https://github.com/AnalogJ/scrutiny/blob/master/README.md
- **Releases**: https://github.com/AnalogJ/scrutiny/releases

### Community Support
- **GitHub Issues**: https://github.com/AnalogJ/scrutiny/issues
- **Discussions**: https://github.com/AnalogJ/scrutiny/discussions
- **Reddit**: r/selfhosted

## 12. Contributing

### How to Contribute
1. Fork the repository on GitHub
2. Create a feature branch
3. Submit pull request
4. Follow Go and Angular coding standards
5. Include tests and documentation

### Development Setup
```bash
# Clone repository
git clone https://github.com/AnalogJ/scrutiny.git
cd scrutiny

# Install dependencies
go mod download
npm install

# Build frontend
npm run build

# Build backend
go build
```

## 13. License

Scrutiny is licensed under the MIT License.

Key points:
- Free to use, modify, and distribute
- Commercial use allowed
- No warranty provided
- Attribution required

## 14. Acknowledgments

### Credits
- **AnalogJ**: Creator and primary developer
- **Community Contributors**: Feature development and testing
- **smartmontools**: S.M.A.R.T. data collection foundation
- **InfluxDB**: Time-series database support

## 15. Version History

### Recent Releases
- **v0.7.x**: Latest stable with enhanced UI
- **v0.6.x**: Added notification support
- **v0.5.x**: Improved collector architecture

### Major Features by Version
- **v0.7**: Enhanced dashboard, better device detection
- **v0.6**: Multiple notification channels
- **v0.5**: Collector/dashboard separation

## 16. Appendices

### A. SMART Attribute Reference

Common SMART attributes monitored:
- **01**: Read Error Rate
- **05**: Reallocated Sectors Count
- **09**: Power-On Hours
- **0A**: Spin Retry Count
- **0C**: Power Cycle Count
- **C4**: Reallocation Event Count
- **C5**: Current Pending Sector Count
- **C6**: Uncorrectable Sector Count

### B. API Usage Examples

```bash
# Get device list
curl http://localhost:8080/api/devices

# Get device details
curl http://localhost:8080/api/device/{device_id}

# Force metrics collection
curl -X POST http://localhost:8080/api/device/{device_id}/smart/scan

# Get device smart data
curl http://localhost:8080/api/device/{device_id}/smart
```

### C. Integration Examples

#### Prometheus Metrics Export
```yaml
# Add to docker-compose.yml
services:
  scrutiny:
    environment:
      - SCRUTINY_WEB_INFLUXDB_INIT_USERNAME=admin
      - SCRUTINY_WEB_INFLUXDB_INIT_PASSWORD=password123
      - SCRUTINY_WEB_INFLUXDB_INIT_ORG=scrutiny
      - SCRUTINY_WEB_INFLUXDB_INIT_BUCKET=metrics
```

### D. Multi-Host Setup

```yaml
# Central collector configuration
api:
  endpoint: "http://scrutiny-central:8080"

# Host-specific collector setup
host:
  id: "storage-server-1"
```

---

For more information and updates, visit https://github.com/howtomgr/scrutiny