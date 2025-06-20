<<<<<<<<<<<<<<<<<<   Fleet Installation Guide    >>>>>>>>>>>>>>>>>>>>>>>>
A comprehensive guide for installing and configuring Fleet MDM (Mobile Device Management) on Ubuntu Server using Docker.

## Table of Contents
- [Prerequisites](#prerequisites)
- [System Requirements](#system-requirements)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Starting Fleet Services](#starting-fleet-services)
- [Agent Deployment](#agent-deployment)
- [Management Commands](#management-commands)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)

## Prerequisites

- Ubuntu Server (18.04 LTS or later recommended)
- Root or sudo access
- Minimum 4GB RAM, 20GB disk space
- Network connectivity for downloading packages

## System Requirements

- **RAM**: 4GB minimum, 8GB recommended
- **Storage**: 20GB minimum, 50GB recommended
- **Network**: Open ports 8080 (Fleet UI), 3306 (MySQL), 6379 (Redis)

## Installation Steps

### 1. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Dependencies

```bash
sudo apt install -y git curl wget software-properties-common apt-transport-https ca-certificates gnupg lsb-release
```

### 3. Install Docker (required for Fleet)

Add Docker GPG key and repository:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package index and install Docker:
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 4. Add Current User to Docker Group (optional)

```bash
sudo usermod -aG docker $USER
```
> **Note**: You'll need to logout and login again for this to take effect

### 5. Install Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 6. Clone the Fleet Repository

```bash
git clone https://github.com/fleetdm/fleet.git
cd fleet
```

### 7. Copy and Configure the Environment File

```bash
cp .env.example .env
```

### 8. Edit the .env File with Your Configuration

You'll need to modify database credentials, server URL, etc.

**Create and edit the .env file:**
```bash
nano .env
```

## Configuration

### Complete .env File Configuration

Create the `.env` file with the following content:

```bash
cat > .env << 'EOF'
# Fleet Server Configuration
FLEET_SERVER_ADDRESS=0.0.0.0:8080
FLEET_SERVER_URL=http://localhost:8080
FLEET_LOGGING_JSON=true

# Database Configuration (MySQL)
FLEET_MYSQL_ADDRESS=mysql:3306
FLEET_MYSQL_DATABASE=fleet
FLEET_MYSQL_USERNAME=fleet
FLEET_MYSQL_PASSWORD=fleet_password_change_me
FLEET_MYSQL_ROOT_PASSWORD=root_password_change_me

# Redis Configuration
FLEET_REDIS_ADDRESS=redis:6379

# JWT Secret (change this to a random string)
FLEET_AUTH_JWT_KEY=your_jwt_secret_key_change_me

# Session Configuration
FLEET_SESSION_KEY=your_session_key_change_me

# File Storage (for software installers, etc.)
FLEET_S3_BUCKET=
FLEET_S3_PREFIX=
FLEET_S3_ACCESS_KEY_ID=
FLEET_S3_SECRET_ACCESS_KEY=
FLEET_S3_STS_ASSUME_ROLE_ARN=
FLEET_S3_ENDPOINT_URL=

# Email Configuration (SMTP)
FLEET_SMTP_SERVER=
FLEET_SMTP_PORT=587
FLEET_SMTP_AUTHENTICATION_TYPE=authtype_username_password
FLEET_SMTP_USERNAME=
FLEET_SMTP_PASSWORD=
FLEET_SMTP_SENDER_ADDRESS=
FLEET_SMTP_ENABLE_SSL_TLS=true
FLEET_SMTP_AUTHENTICATION_METHOD=authmethod_plain
FLEET_SMTP_DOMAIN=
FLEET_SMTP_VERIFY_SSL_CERTS=true
FLEET_SMTP_ENABLE_START_TLS=true

# Osquery Configuration
FLEET_OSQUERY_RESULT_LOG_FILE=/tmp/osquery_result
FLEET_OSQUERY_STATUS_LOG_FILE=/tmp/osquery_status

# Vulnerability Processing
FLEET_VULNERABILITIES_DATABASES_PATH=/tmp/vuln

# License (for Fleet Premium features)
FLEET_LICENSE_KEY=

# Development/Debug
FLEET_DEV_MODE=false
FLEET_LOGGING_DEBUG=false

# Additional Security
FLEET_SERVER_TLS=false
FLEET_SERVER_CERT=
FLEET_SERVER_KEY=

# Webhook settings
FLEET_WEBHOOK_SETTINGS_VULNERABILITIES_WEBHOOK_URL=
FLEET_WEBHOOK_SETTINGS_HOST_STATUS_WEBHOOK_URL=

EOF
```

### Make the Configuration File Readable

```bash
chmod 644 .env
```

### Display Configuration Success Message

```bash
echo "✅ .env file created successfully!"
echo "⚠️  IMPORTANT: Please edit the .env file and change the default passwords and secrets:"
echo "   - FLEET_MYSQL_PASSWORD"
echo "   - FLEET_MYSQL_ROOT_PASSWORD" 
echo "   - FLEET_AUTH_JWT_KEY"
echo "   - FLEET_SESSION_KEY"
echo "   - FLEET_SERVER_URL (set to your server's IP/domain)"
echo ""
echo "Edit the file with: nano .env"
```

### ⚠️ **CRITICAL SECURITY CHANGES REQUIRED**

**Edit the .env file and change the following:**
```bash
nano .env
```

**Change these values:**
- `FLEET_MYSQL_PASSWORD` to a strong password
- `FLEET_MYSQL_ROOT_PASSWORD` to a strong password
- `FLEET_AUTH_JWT_KEY` to a random 32+ character string
- `FLEET_SESSION_KEY` to a random 32+ character string
- Update `FLEET_SERVER_URL` to your server's IP address (e.g., `http://192.168.1.100:8080`)

### 9. Install Node.js Dependencies

```bash
make deps-js
make generate-js
make generate-go
npm install -g yarn
```

### Verify Yarn Installation

```bash
yarn --version
```

## Starting Fleet Services

### 10. Start Fleet Using Docker Compose

```bash
docker-compose up -d
```

### 11. Check if Services are Running

```bash
docker-compose ps
```

### 12. View Logs

```bash
docker-compose logs -f
```

### 13. Access Fleet Web Interface

- **Default URL**: `http://your-server-ip:8080`
- **Example**: `https://192.168.1.116:8080/`
- Default admin credentials are typically created during first setup

## Management Commands

### Additional Commands for Management

#### Stop Fleet Services
```bash
docker-compose down
```

#### Restart Fleet Services
```bash
docker-compose restart
```

#### Update Fleet (pull latest changes and rebuild)
```bash
git pull
docker-compose down
docker-compose up -d --build
```

#### View Specific Service Logs
```bash
# View Fleet service logs
docker-compose logs fleet

# View MySQL service logs
docker-compose logs mysql

# View Redis service logs
docker-compose logs redis
```

## System Service Management

### Stop Conflicting Services

#### Stop Both MySQL and Redis System Services
```bash
sudo systemctl stop mysql redis-server
sudo systemctl disable mysql redis-server
```

#### Stop MySQL Service Only
```bash
sudo systemctl stop mysql
sudo systemctl disable mysql
```

#### Check Network Ports
```bash
sudo netstat -tlnp
```

#### Check Redis Service Status
```bash
sudo systemctl status redis
sudo systemctl status redis-server
```

#### Stop Redis if Running as System Service
```bash
sudo systemctl stop redis
```

## Agent Deployment

### Fleet Agent for Windows

#### Download Fleet Tools
Download the file from [Fleet Releases](https://github.com/fleetdm/fleet/releases) (`.zip` or `.tar.gz`)

#### Installation Steps

1. **Extract Files**: Download and install to `C:\fleet` directory
2. **Run Powershell as Administrator**: Navigate to `C:\fleet` path
3. **Generate MSI Package**: Run the following command as Administrator:

```cmd
C:\fleet>fleetctl package --type=msi --enable-scripts --fleet-url=https://192.168.1.116:8080 --enroll-secret=b51LHW1Vik+FoCdaYzeaerX5cxYIMTEi --insecure
```

#### Expected Result
The command creates the `fleet-osquery.msi` file with output similar to:

```
Generating your fleetd agent...
Windows Installer XML Toolset Toolset Harvester version
Copyright (c) .NET Foundation and contributors. All rights reserved.

Windows Installer XML Toolset Compiler version
Copyright (c) .NET Foundation and contributors. All rights reserved.

heat.wxs
main.wxs
Windows Installer XML Toolset Linker version
Copyright (c) .NET Foundation and contributors. All rights reserved.

Success! You generated fleetd at C:\fleet\fleet-osquery.msi

To add hosts to Fleet, install fleetd.
Learn how: https://fleetdm.com/learn-more-about/enrolling-hosts
```

## Alternative: Running Fleet Binary

### To Run Fleet Server Binary

#### Step 1: Set Environment Variables
```bash
export FLEET_MYSQL_USERNAME=fleet
export FLEET_MYSQL_PASSWORD=insecure
export FLEET_MYSQL_DATABASE=fleet
export FLEET_MYSQL_ADDRESS=127.0.0.1:3306  # or use 'mysql:3306' for Docker service
```

#### Step 2: Start Fleet Server
```bash
./build/fleet serve
```

## Troubleshooting

### Common Issues and Solutions

#### Port Conflicts
Check if ports are already in use:
```bash
sudo netstat -tlnp | grep -E ':(8080|3306|6379)'
```

#### Service Status Checks
```bash
# Check all Docker containers
docker ps -a

# Check specific Fleet services
docker-compose ps

# Check system services
sudo systemctl status mysql
sudo systemctl status redis-server
```

#### Database Connection Issues
```bash
# Test MySQL connection
mysql -h 127.0.0.1 -P 3306 -u fleet -p

# Check MySQL logs
docker-compose logs mysql
```

#### Permission Issues
```bash
# Verify user is in docker group
groups $USER

# Fix docker socket permissions
sudo chmod 666 /var/run/docker.sock
```

### Log Analysis

#### Fleet Application Logs
```bash
# Real-time Fleet logs
docker-compose logs -f fleet

# Last 100 lines of Fleet logs
docker-compose logs --tail=100 fleet
```

#### Database and Cache Logs
```bash
# MySQL logs
docker-compose logs mysql

# Redis logs  
docker-compose logs redis
```

#### System Resource Monitoring
```bash
# Check disk space
df -h

# Check memory usage
free -h

# Check running processes
top
```

## Security Considerations

### Essential Security Steps

1. **Change Default Passwords**: 
   - MySQL passwords in `.env` file
   - Generate strong JWT and session keys

2. **Network Security**:
   - Configure firewall rules
   - Use HTTPS in production
   - Restrict access to management ports

3. **File Permissions**:
   ```bash
   # Secure .env file
   chmod 600 .env
   chown $USER:$USER .env
   ```

4. **Regular Updates**:
   ```bash
   # Update system packages
   sudo apt update && sudo apt upgrade -y
   
   # Update Fleet
   git pull
   docker-compose pull
   docker-compose up -d
   ```

### Production Deployment Checklist

- [ ] Change all default passwords and secrets
- [ ] Configure HTTPS/TLS certificates
- [ ] Set up proper firewall rules
- [ ] Configure backup strategy for database
- [ ] Set up monitoring and alerting
- [ ] Configure SMTP for email notifications
- [ ] Test agent enrollment process
- [ ] Document custom configurations

## Additional Resources

- [Fleet Documentation](https://fleetdm.com/docs)
- [Fleet GitHub Repository](https://github.com/fleetdm/fleet)
- [Fleet Community Slack](https://fleetdm.com/slack)
- [Osquery Documentation](https://osquery.readthedocs.io/)

## Network Configuration

### Required Ports

| Port | Service | Purpose |
|------|---------|---------|
| 8080 | Fleet UI/API | Web interface and API access |
| 3306 | MySQL | Database connections |
| 6379 | Redis | Cache and session storage |

### Firewall Configuration Example

```bash
# Allow Fleet web interface
sudo ufw allow 8080/tcp

# Allow SSH (if needed)
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

---

**Version**: 1.0  
**Last Updated**: June 2025  
**Compatibility**: Fleet v4.x, Ubuntu 18.04+  
**Tested Environment**: Ubuntu Server 20.04 LTS with Docker 24.x
