# IKFlix: Your Personal Streaming Platform
A complete guide to deploy your own streaming service with professional monitoring.

## What is IKFlix?
IKFlix is a personal streaming platform built on top of Jellyfin that lets you:
- 🎬 Stream your media collection anywhere
- 📺 Watch live TV channels
- 📊 Monitor your service performance
- 👥 Manage multiple users
- 📱 Support multiple devices

## System Requirements
Minimum requirements for a basic setup (up to 5 concurrent users):
- CPU: 4 cores
- RAM: 8GB
- Storage: 50GB + media storage
- Network: 100Mbps upload

Recommended for larger setup (20+ users):
- CPU: 8+ cores
- RAM: 16GB
- Storage: 100GB + media storage
- Network: 1Gbps upload

## Project Structure
```
/ikflix/
├── config/             # Main configuration
├── media/             # Media storage
│   ├── movies/
│   ├── shows/
│   └── live/
├── monitoring/        # Monitoring setup
├── transcodes/        # Temporary transcoding
└── docker/            # Docker configurations
```

## Step-by-Step Deployment

### 1. Initial Server Setup

```bash
# Create base directory structure
sudo mkdir -p /ikflix/{config,media,monitoring,transcodes,docker}
sudo mkdir -p /ikflix/media/{movies,shows,live}
sudo mkdir -p /ikflix/docker/monitoring

# Set permissions
sudo chown -R $USER:$USER /ikflix
```

### 2. Environment Configuration

Create `/ikflix/docker/.env`:
```bash
# IKFlix Configuration
IKFLIX_DOMAIN=your-domain.com
JELLYFIN_PORT=8096
ADMIN_PASSWORD=your-secure-password

# Monitoring Configuration
GRAFANA_PORT=3000
PROMETHEUS_PORT=9090
MONITOR_PASSWORD=your-monitor-password

# Storage Configuration
MEDIA_PATH=/ikflix/media
CONFIG_PATH=/ikflix/config
TRANSCODE_PATH=/ikflix/transcodes
```

### 3. Main Service Configuration

Create `/ikflix/docker/docker-compose.yml`:
```yaml
version: '3.8'

networks:
  ikflix_network:
    driver: bridge

services:
  # Main Streaming Service
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: ikflix_streaming
    restart: unless-stopped
    networks:
      - ikflix_network
    ports:
      - "${JELLYFIN_PORT}:8096"
    environment:
      - JELLYFIN_PublishedServerUrl=http://${IKFLIX_DOMAIN}
      - JELLYFIN_CACHE_DIR=/cache
    volumes:
      - ${CONFIG_PATH}:/config
      - ${MEDIA_PATH}:/media:ro
      - ${TRANSCODE_PATH}:/transcodes
    devices:
      - /dev/dri:/dev/dri  # Hardware acceleration (if available)

  # Monitoring Stack
  prometheus:
    image: prom/prometheus:latest
    container_name: ikflix_prometheus
    restart: unless-stopped
    networks:
      - ikflix_network
    ports:
      - "${PROMETHEUS_PORT}:9090"
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    container_name: ikflix_grafana
    restart: unless-stopped
    networks:
      - ikflix_network
    ports:
      - "${GRAFANA_PORT}:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${MONITOR_PASSWORD}
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
    volumes:
      - ./monitoring/grafana:/etc/grafana
      - grafana_data:/var/lib/grafana

  node_exporter:
    image: prom/node-exporter:latest
    container_name: ikflix_node_exporter
    restart: unless-stopped
    networks:
      - ikflix_network
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

volumes:
  prometheus_data:
  grafana_data:
```

### 4. Monitoring Configuration

Create Prometheus config `/ikflix/docker/monitoring/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'ikflix_system'
    static_configs:
      - targets: ['node_exporter:9100']
        labels:
          service: 'ikflix'

  - job_name: 'ikflix_streaming'
    static_configs:
      - targets: ['jellyfin:8096']
        labels:
          service: 'ikflix'
```

### 5. Service Deployment

```bash
# Navigate to docker directory
cd /ikflix/docker

# Start the services
docker-compose up -d

# Check service status
docker-compose ps
```

## Post-Deployment Setup

### 1. Initial Jellyfin Configuration

1. Access your server: `http://your-domain:8096`
2. Complete the setup wizard:
   - Create admin account
   - Add media libraries
   - Configure transcoding settings

### 2. Media Library Setup

Add your media following this structure:
```
/ikflix/media/
├── movies/
│   ├── Movie1 (2023)/
│   │   └── Movie1.mkv
│   └── Movie2 (2024)/
│       └── Movie2.mp4
└── shows/
    └── Show1/
        └── Season 01/
            ├── S01E01.mkv
            └── S01E02.mkv
```

### 3. Monitoring Setup

1. Access Grafana: `http://your-domain:3000`
2. Login with:
   - Username: admin
   - Password: (from .env file)
3. Add Prometheus data source:
   - URL: http://prometheus:9090
   - Access: Browser

### 4. IPTV Integration (Optional)

For live TV support:
1. Create `/ikflix/media/live/channels.m3u`
2. Add your M3U content
3. In Jellyfin:
   - Go to Live TV
   - Add M3U tuner
   - Point to `/media/live/channels.m3u`

## Administration Guide

### User Management

1. Creating Users:
   - Dashboard → Users → New User
   - Set permissions
   - Configure quality settings

2. Library Management:
   - Regular content updates
   - Metadata management
   - Storage optimization

### Performance Tuning

1. Transcoding Settings:
```json
{
  "EnableHardwareEncoding": true,
  "EnableThrottling": true,
  "ThrottleDelaySeconds": 3
}
```

2. Network Settings:
```json
{
  "EnableRemoteAccess": true,
  "RemoteClientBitrateLimit": 20000000
}
```

### Monitoring Best Practices

1. Daily Checks:
   - Active users
   - System resources
   - Streaming quality

2. Weekly Tasks:
   - Update media libraries
   - Check transcoding stats
   - Review user activity

3. Monthly Maintenance:
   - System updates
   - Backup configurations
   - Performance optimization

## Troubleshooting

### Common Issues

1. Streaming Issues:
```bash
# Check Jellyfin logs
docker-compose logs jellyfin

# Verify network settings
netstat -tulpn | grep 8096
```

2. Monitoring Issues:
```bash
# Check Prometheus targets
curl localhost:9090/targets

# Verify Grafana connection
docker-compose logs grafana
```

3. Performance Issues:
- Review hardware transcoding
- Check network bandwidth
- Monitor system resources

## Backup and Recovery

### Regular Backups

```bash
# Backup script
#!/bin/bash
BACKUP_DIR="/backup/ikflix"
DATE=$(date +%Y%m%d)

# Backup configurations
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /ikflix/config/

# Backup monitoring data
docker-compose exec prometheus /bin/prometheus-backup
```

### Recovery Process

```bash
# Restore configurations
tar -xzf config_backup.tar.gz -C /ikflix/config/

# Restart services
docker-compose restart
```

## Upgrading

```bash
# Pull new images
docker-compose pull

# Update services
docker-compose up -d

# Check logs
docker-compose logs -f
```

## Security Recommendations

1. Basic Security:
   - Use strong passwords
   - Enable HTTPS
   - Regular updates

2. Advanced Security:
   - Implement reverse proxy
   - Configure firewall rules
   - Enable authentication

