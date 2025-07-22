# Fleet MDM Complete Installation Guide for Ubuntu Server

> ⚠️ **IMPORTANT**: This guide must be followed step-by-step without skipping any commands. Each step builds upon the previous one.

## Overview
Fleet is an open-source device management platform built on osquery. This guide provides a complete installation process for Ubuntu Server that will result in a fully functional Fleet MDM system.

## Prerequisites (MANDATORY)
- Ubuntu Server 18.04 LTS or later
- Root or sudo access
- Minimum 4GB RAM, 20GB disk space
- Network connectivity for downloading packages
- Server with static IP or proper DNS configuration

## Estimated Installation Time: 90-120 minutes

---

## 📋 Pre-Installation Checklist

Before starting, verify these requirements:

- [ ] Server has minimum 4GB RAM and 20GB free disk space
- [ ] You have sudo/root access
- [ ] Ports 8080, 3306, and 6379 are available
- [ ] Server has internet connectivity
- [ ] You have planned your admin credentials

---

## Phase 1: Infrastructure Setup (30-45 minutes)

### Step 1: System Update and Essential Packages (REQUIRED)

```bash
# Update system packages - DO NOT SKIP
sudo apt update && sudo apt upgrade -y

# Install essential packages - ALL REQUIRED
sudo apt install -y git curl wget software-properties-common apt-transport-https ca-certificates gnupg lsb-release unzip

# Verify installation
dpkg -l | grep -E "(git|curl|wget)"
```

**✅ Verification**: Ensure all packages installed without errors.

### Step 2: Docker Installation (CRITICAL)

```bash
# Add Docker GPG key - REQUIRED FOR SECURITY
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository - EXACT COMMAND REQUIRED
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index after adding repository
sudo apt update

# Install Docker components - ALL REQUIRED
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add current user to docker group - REQUIRED FOR PERMISSIONS
sudo usermod -aG docker $USER

# Start and enable Docker service - REQUIRED
sudo systemctl start docker
sudo systemctl enable docker

# Verify Docker installation - MUST SHOW VERSION
docker --version
```

**✅ Verification**: Docker version should display. If not, troubleshoot before proceeding.

**⚠️ IMPORTANT**: Log out and log back in for group changes to take effect, or run:
```bash
newgrp docker
```

### Step 3: Docker Compose Installation (MANDATORY)

```bash
# Install Docker Compose - PRIMARY METHOD
sudo apt install -y docker-compose

# If above fails, use alternative method
if ! command -v docker-compose &> /dev/null; then
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
fi

# Verify Docker Compose installation - MUST SHOW VERSION
docker-compose --version
```

**✅ Verification**: Docker Compose version should display.

### Step 4: Fleet Directory Setup (REQUIRED)

```bash
# Create Fleet working directory
sudo mkdir -p /opt/fleet
sudo chown $USER:$USER /opt/fleet
cd /opt/fleet

# Verify current directory
pwd
# Should output: /opt/fleet
```

### Step 5: Stop Conflicting Services (CRITICAL)

```bash
# Stop conflicting MySQL services - PREVENTS PORT CONFLICTS
sudo systemctl stop mysql mysql.service mariadb mariadb.service 2>/dev/null || true
sudo systemctl disable mysql mysql.service mariadb mariadb.service 2>/dev/null || true

# Stop conflicting Redis services - PREVENTS PORT CONFLICTS
sudo systemctl stop redis redis-server 2>/dev/null || true
sudo systemctl disable redis redis-server 2>/dev/null || true

# Check for port conflicts - MUST BE CLEAR
echo "Checking for port conflicts..."
sudo netstat -tlnp | grep -E ':(8080|3306|6379)' || echo "Ports are available"
```

**✅ Verification**: No output from netstat command means ports are available.

---

## Phase 2: Fleet Configuration (20-30 minutes)

### Step 6: Create Docker Compose Configuration (MOST CRITICAL)

```bash
# Create docker-compose.yml - DO NOT MODIFY CONTENT
cat > /opt/fleet/docker-compose.yml << 'EOF'
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: fleet_mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${FLEET_MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${FLEET_MYSQL_DATABASE}
      MYSQL_USER: ${FLEET_MYSQL_USERNAME}
      MYSQL_PASSWORD: ${FLEET_MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - fleet-network
    command: --default-authentication-plugin=mysql_native_password

  redis:
    image: redis:7-alpine
    container_name: fleet_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - fleet-network

  fleet:
    image: fleetdm/fleet:latest
    container_name: fleet_server
    restart: unless-stopped
    depends_on:
      - mysql
      - redis
    environment:
      FLEET_MYSQL_ADDRESS: ${FLEET_MYSQL_ADDRESS}
      FLEET_MYSQL_DATABASE: ${FLEET_MYSQL_DATABASE}
      FLEET_MYSQL_USERNAME: ${FLEET_MYSQL_USERNAME}
      FLEET_MYSQL_PASSWORD: ${FLEET_MYSQL_PASSWORD}
      FLEET_REDIS_ADDRESS: ${FLEET_REDIS_ADDRESS}
      FLEET_SERVER_ADDRESS: ${FLEET_SERVER_ADDRESS}
      FLEET_AUTH_JWT_KEY: ${FLEET_AUTH_JWT_KEY}
      FLEET_SESSION_KEY: ${FLEET_SESSION_KEY}
      FLEET_SERVER_CERT: ${FLEET_SERVER_CERT}
      FLEET_SERVER_KEY: ${FLEET_SERVER_KEY}
      FLEET_SERVER_TLS: ${FLEET_SERVER_TLS}
      FLEET_LOGGING_JSON: ${FLEET_LOGGING_JSON}
    ports:
      - "8080:8080"
    volumes:
      - fleet_data:/tmp
    networks:
      - fleet-network

volumes:
  mysql_data:
  redis_data:
  fleet_data:

networks:
  fleet-network:
    driver: bridge
EOF

# Verify file creation
ls -la /opt/fleet/docker-compose.yml
```

**✅ Verification**: File should exist and be readable.

### Step 7: Create Environment Configuration (CRITICAL)

⚠️ **SECURITY WARNING**: Change the default passwords before production use!

```bash
# Create .env file with secure configuration
cat > /opt/fleet/.env << 'EOF'
# Fleet Server Configuration
FLEET_SERVER_ADDRESS=0.0.0.0:8080
FLEET_SERVER_URL=http://localhost:8080
FLEET_LOGGING_JSON=true

# Database Configuration (MySQL)
FLEET_MYSQL_ADDRESS=mysql:3306
FLEET_MYSQL_DATABASE=fleet
FLEET_MYSQL_USERNAME=fleet
FLEET_MYSQL_PASSWORD=SecureFleetPassword123!
FLEET_MYSQL_ROOT_PASSWORD=SecureRootPassword123!

# Redis Configuration
FLEET_REDIS_ADDRESS=redis:6379

# Security Keys (CHANGE THESE IN PRODUCTION)
FLEET_AUTH_JWT_KEY=your_very_secure_jwt_secret_key_32_chars_minimum_change_this
FLEET_SESSION_KEY=your_very_secure_session_key_32_chars_minimum_change_this

# TLS Configuration
FLEET_SERVER_TLS=false
FLEET_SERVER_CERT=
FLEET_SERVER_KEY=

# Osquery Configuration
FLEET_OSQUERY_RESULT_LOG_FILE=/tmp/osquery_result
FLEET_OSQUERY_STATUS_LOG_FILE=/tmp/osquery_status

# Vulnerability Processing
FLEET_VULNERABILITIES_DATABASES_PATH=/tmp/vuln

# Development/Debug
FLEET_DEV_MODE=false
FLEET_LOGGING_DEBUG=false
EOF

# Secure the .env file - CRITICAL FOR SECURITY
chmod 600 /opt/fleet/.env

# Set proper ownership
sudo chown $USER:docker /opt/fleet/.env

# Verify file creation and permissions
ls -la /opt/fleet/.env
```

**✅ Verification**: File should show `-rw-------` permissions.

### Step 8: Install Fleet CLI (REQUIRED FOR MANAGEMENT)

```bash
# Download fleetctl CLI tool
cd /tmp
curl -L https://github.com/fleetdm/fleet/releases/latest/download/fleetctl-linux.tar.gz -o fleetctl.tar.gz

# Extract and install
tar -xzf fleetctl.tar.gz
sudo mv fleetctl /usr/local/bin/
sudo chmod +x /usr/local/bin/fleetctl

# Verify installation - MUST SHOW VERSION
fleetctl --version

# Clean up download files
rm fleetctl.tar.gz

# Return to Fleet directory
cd /opt/fleet
```

**✅ Verification**: fleetctl version should display.

---

## Phase 3: Service Deployment (20-30 minutes)

### Step 9: Configure Firewall (SECURITY CRITICAL)

```bash
# Reset firewall to clean state
sudo ufw --force reset

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH - CRITICAL: DON'T LOCK YOURSELF OUT
sudo ufw allow 22/tcp

# Allow Fleet web interface
sudo ufw allow 8080/tcp

# Allow MySQL from private networks only (more secure)
sudo ufw allow from 10.0.0.0/8 to any port 3306
sudo ufw allow from 172.16.0.0/12 to any port 3306  
sudo ufw allow from 192.168.0.0/16 to any port 3306

# Enable firewall
sudo ufw --force enable

# Verify firewall status
sudo ufw status verbose
```

**✅ Verification**: Firewall should be active with correct rules.

### Step 10: Start Fleet Services (CRITICAL)

```bash
# Ensure you're in the correct directory
cd /opt/fleet

# Pull latest images - ENSURES LATEST VERSIONS
docker-compose pull

# Start all services in background
docker-compose up -d

# Wait for services to initialize
echo "Waiting 60 seconds for services to start..."
sleep 60

# Check service status - ALL SHOULD BE "Up"
docker-compose ps
```
# Initialize Fleet database
docker-compose exec fleet fleet prepare db \
  --mysql_address=mysql:3306 \
  --mysql_database=fleet \
  --mysql_username=fleet \
  --mysql_password=fleet


run ' fleet prepare db ' and then run docker-compose up -d 

**✅ Verification**: All containers should show "Up" status.

### Step 11: Verify Service Health (MANDATORY)

```bash
# Check container logs for errors
echo "=== Fleet Server Logs ==="
docker-compose logs --tail=20 fleet

echo "=== MySQL Logs ==="
docker-compose logs --tail=10 mysql

echo "=== Redis Logs ==="
docker-compose logs --tail=10 redis

# Test database connectivity
echo "=== Testing Database Connection ==="
docker-compose exec mysql mysql -u fleet -pSecureFleetPassword123! -e "SELECT 'Database connection successful' AS status;"

# Test Redis connectivity
echo "=== Testing Redis Connection ==="
docker-compose exec redis redis-cli ping

# Check listening ports
echo "=== Checking Port Status ==="
sudo netstat -tlnp | grep -E ':(8080|3306|6379)'
```

**✅ Verification**: 
- Database should return "Database connection successful"
- Redis should return "PONG"
- All ports should show LISTEN status

---

## Phase 4: Database Initialization (CRITICAL)

### Step 12: Initialize Fleet Database (MUST COMPLETE)

```bash
# Wait for MySQL to be fully ready
echo "Ensuring MySQL is ready for database initialization..."
until docker-compose exec mysql mysql -u root -pSecureRootPassword123! -e "SELECT 1" >/dev/null 2>&1; do
  echo "Waiting for MySQL..."
  sleep 5
done

echo "MySQL is ready. Initializing Fleet database..."

# Initialize Fleet database schema
docker-compose exec fleet fleet prepare db \
  --mysql_address=mysql:3306 \
  --mysql_database=fleet \
  --mysql_username=fleet \
  --mysql_password=SecureFleetPassword123!

# Check initialization status
if [ $? -eq 0 ]; then
    echo "✅ Database initialization completed successfully"
else
    echo "❌ Database initialization failed - check logs"
    docker-compose logs fleet
fi
```

**✅ Verification**: Should show "Database initialization completed successfully".

### Step 13: Verify Fleet API (FINAL CHECK)

```bash
# Test Fleet API health
echo "Testing Fleet API..."
curl -s -I http://localhost:8080/api/v1/fleet/version

# Get server IP for external access
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "Fleet is accessible at:"
echo "  Internal: http://localhost:8080"
echo "  External: http://$SERVER_IP:8080"

# Final container status check
echo "=== Final Container Status ==="
docker-compose ps
```

**✅ Verification**: API should return HTTP 200 status.

---

## Phase 5: Initial Setup (15-20 minutes)

### Step 14: Access Fleet Web Interface

1. **Open your web browser** and navigate to:
   - `http://your-server-ip:8080` (replace with actual IP)
   - Or `http://localhost:8080` if accessing locally

2. **Complete the setup wizard**:
   - Create admin account
   - Set organization name
   - Configure basic settings

3. **Test login** with your admin credentials

---

## Post-Installation Verification Checklist

Run these commands to ensure everything is working:

```bash
# Complete system check
cd /opt/fleet

echo "=== Docker Status ==="
docker --version
docker-compose --version

echo "=== Container Status ==="
docker-compose ps

echo "=== Service Health ==="
docker-compose exec mysql mysql -u fleet -pSecureFleetPassword123! -e "SHOW DATABASES;"
docker-compose exec redis redis-cli ping
curl -s -I http://localhost:8080/api/v1/fleet/version

echo "=== Port Status ==="
sudo netstat -tlnp | grep -E ':(8080|3306|6379)'

echo "=== Fleet CLI ==="
fleetctl --version

echo "=== Firewall Status ==="
sudo ufw status
```

**✅ All checks should pass before proceeding to use Fleet.**

---

## Daily Management Commands

### Starting/Stopping Services
```bash
# Start services
cd /opt/fleet
docker-compose up -d

# Stop services
docker-compose down

# Restart services
docker-compose restart

# View real-time logs
docker-compose logs -f fleet
```

### Backup Commands
```bash
# Backup database
docker-compose exec mysql mysqldump -u root -pSecureRootPassword123! fleet > fleet_backup_$(date +%Y%m%d).sql

# Backup configuration
tar -czf fleet_config_backup_$(date +%Y%m%d).tar.gz /opt/fleet/.env /opt/fleet/docker-compose.yml
```

### Update Fleet
```bash
cd /opt/fleet
docker-compose pull
docker-compose down
docker-compose up -d
```

---

## Troubleshooting

### Common Issues and Solutions

**Issue**: Containers won't start
```bash
# Check logs
docker-compose logs

# Check port conflicts
sudo netstat -tlnp | grep -E ':(8080|3306|6379)'

# Restart services
docker-compose down
docker-compose up -d
```

**Issue**: Database connection failed
```bash
# Check MySQL container
docker-compose logs mysql

# Reset database
docker-compose down -v
docker-compose up -d
# Wait and re-run database initialization
```

**Issue**: Web interface not accessible
```bash
# Check firewall
sudo ufw status

# Check Fleet container
docker-compose logs fleet

# Test local connectivity
curl http://localhost:8080/api/v1/fleet/version
```

---

## Security Hardening (Post-Installation)

```bash
# Set proper file permissions
sudo chown -R $USER:docker /opt/fleet
chmod 600 /opt/fleet/.env
chmod 644 /opt/fleet/docker-compose.yml

# Configure log rotation
sudo tee /etc/logrotate.d/fleet << EOF
/var/lib/docker/containers/*/*-json.log {
    rotate 7
    daily
    compress
    size 50M
    missingok
    delaycompress
    copytruncate
}
EOF

# Enable automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## Important Security Notes

⚠️ **BEFORE PRODUCTION USE**:

1. **Change default passwords** in `/opt/fleet/.env`
2. **Generate strong JWT and session keys** (32+ characters)
3. **Configure SSL/TLS certificates** for HTTPS
4. **Restrict database access** to specific networks
5. **Enable backup procedures**
6. **Monitor logs regularly**

---

## Support and Documentation

- **Fleet Documentation**: https://fleetdm.com/docs
- **GitHub Issues**: https://github.com/fleetdm/fleet/issues
- **Community Slack**: https://fleetdm.com/slack

---

## Installation Complete! ✅

If you've followed all steps without skipping any commands, you now have a fully functional Fleet MDM system running on your Ubuntu server.

**Next Steps**:
1. Access the web interface and complete the setup wizard
2. Install osquery agents on your devices
3. Configure your first queries and policies
4. Set up monitoring and alerting

**Total Installation Time**: Approximately 90-120 minutes
