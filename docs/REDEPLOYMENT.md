# Re-deployment Guide

Guide for updating and re-deploying your ESC Django application.

## Table of Contents

- [Overview](#overview)
- [Configuration Persistence](#configuration-persistence)
- [Re-running the Installer](#re-running-the-installer)
- [Update Strategies](#update-strategies)
- [Common Scenarios](#common-scenarios)
- [Troubleshooting](#troubleshooting)

## Overview

The deployment script intelligently detects existing installations and:
- ✅ Preserves your configuration (domain, Docker username, SSL settings)
- ✅ Keeps your environment variables (.env.docker)
- ✅ Skips already-configured components (firewall, user creation)
- ✅ Only asks for what's needed (Docker password for security)
- ✅ Allows overriding any setting if desired

## Configuration Persistence

### What Gets Saved

The script saves these settings to `.deployment_config`:
```bash
DOMAIN_NAME="example.com"
DOCKER_USERNAME="your-username"
APP_DIR="/opt/apps/esc"
SETUP_SSL="letsencrypt"
SSL_EMAIL="admin@example.com"
```

**Location**: `/opt/apps/esc/.deployment_config`

### What Doesn't Get Saved

For security reasons, these are NOT saved:
- ❌ Docker Hub password
- ❌ Environment variables (.env.docker)
- ❌ SSL private keys

You'll be prompted for the Docker password each time, but everything else is remembered.

### What Gets Preserved

When re-deploying:
- ✅ `.env.docker` - Your environment configuration
- ✅ SSL certificates (Let's Encrypt or self-signed)
- ✅ Nginx configuration
- ✅ Docker volumes (redis data, logs, etc.)
- ✅ Management scripts
- ✅ Systemd service

## Re-running the Installer

### Quick Update (Use Existing Config)

**Scenario**: Just want to pull latest code and restart

```bash
cd /opt/apps/esc
./deploy.sh

# You'll see:
# ✓ Existing deployment detected!
#
# Previous configuration:
#   Domain: example.com
#   Docker Hub User: yourusername
#   App Directory: /opt/apps/esc
#   SSL: letsencrypt
#
# Use existing configuration? [Y/n]: Y

# Only asks for:
# - Docker Hub password (security)

# Then proceeds to:
# - Pull latest code
# - Update containers
# - Restart services
```

### Full Reconfiguration

**Scenario**: Want to change domain, SSL, or other settings

```bash
cd /opt/apps/esc
./deploy.sh

# At the prompt:
Use existing configuration? [Y/n]: n

# Now you can reconfigure:
# - New domain name
# - Different SSL option
# - Different Docker username
# etc.
```

### What Happens During Re-deployment

```
┌─────────────────────────────────────┐
│ Detect Existing Installation       │
├─────────────────────────────────────┤
│ ✓ Found .deployment_config          │
│ ✓ Found .env.docker                 │
│ ✓ Found SSL certificates            │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ Ask: Use Existing Config?           │
├─────────────────────────────────────┤
│ Yes → Skip most questions           │
│ No  → Full configuration            │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ Update Components                   │
├─────────────────────────────────────┤
│ ✓ Pull latest code from GitHub     │
│ ✓ Login to Docker Hub               │
│ ⊗ Skip .env (already exists)        │
│ ⊗ Skip SSL (already exists)         │
│ ⊗ Skip firewall (already setup)     │
│ ⊗ Skip user creation (exists)       │
│ ✓ Update Nginx config               │
│ ✓ Update management scripts         │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ Restart Application                 │
├─────────────────────────────────────┤
│ ✓ Pull latest Docker images         │
│ ✓ Restart containers                │
│ ✓ Verify health                     │
└─────────────────────────────────────┘
```

## Update Strategies

### Strategy 1: Simple Code Update

**When to use**: Just updating application code

```bash
cd /opt/apps/esc
./deploy.sh
# Press Y to use existing config
# Enter Docker password
# Done!
```

**Time**: ~2-3 minutes  
**Downtime**: ~30 seconds during restart

### Strategy 2: Zero-Downtime with Blue-Green

**When to use**: Production with no downtime tolerance

```bash
# Not yet implemented in script
# Use Kubernetes or Docker Swarm for true zero-downtime
```

### Strategy 3: Configuration Update

**When to use**: Changing environment variables only

```bash
cd /opt/apps/esc
./reconfig.sh
# Edit .env.docker
# Restart automatically
```

**Time**: ~1 minute  
**Downtime**: ~30 seconds during restart

### Strategy 4: Full Re-deployment

**When to use**: Major changes (domain, SSL, infrastructure)

```bash
cd /opt/apps/esc
./deploy.sh
# Choose "n" to reconfigure
# Answer all questions
# Full setup runs
```

**Time**: ~5-10 minutes  
**Downtime**: ~1 minute during Nginx/service restart

## Common Scenarios

### Scenario 1: Update Application Code

**Situation**: New features pushed to GitHub

```bash
cd /opt/apps/esc
./deploy.sh

# At prompt:
Use existing configuration? [Y/n]: Y
Docker Hub password: ••••••••

# Script will:
# 1. Git pull latest code ✓
# 2. Pull latest Docker images ✓
# 3. Restart containers ✓
```

**Alternative (faster)**:
```bash
cd /opt/apps/esc
./deploy.sh  # Use management script directly
# Pulls image and restarts
```

### Scenario 2: Change Domain Name

**Situation**: Moving from test.example.com to example.com

```bash
cd /opt/apps/esc
./deploy.sh

# At prompt:
Use existing configuration? [Y/n]: n
Enter your domain name [test.example.com]: example.com

# Update environment file:
nano .env.docker
# Change ALLOWED_HOSTS, SITE_URL, etc.
# Save and continue

# Script will:
# 1. Update Nginx config with new domain ✓
# 2. If using Let's Encrypt, obtain new cert ✓
# 3. Update all configurations ✓
```

**Don't forget**:
- Update DNS records
- Update Cloudflare settings
- Update .env.docker with new domain

### Scenario 3: Switch SSL Type

**Situation**: Had no SSL, now want Let's Encrypt

```bash
cd /opt/apps/esc
./deploy.sh

# At prompt:
Use existing configuration? [Y/n]: n
# Keep same domain, username, etc.

# At SSL prompt:
Keep existing SSL configuration? [Y/n]: n
Select option [1/2/3]: 1  # Let's Encrypt
Email: admin@example.com

# Script will:
# 1. Install Certbot ✓
# 2. Obtain SSL certificate ✓
# 3. Update Nginx for HTTPS ✓
# 4. Setup auto-renewal ✓
```

**Update Cloudflare**:
- Change SSL mode to "Full (strict)"

### Scenario 4: Update Environment Variables

**Situation**: Need to change API keys, database URL, etc.

```bash
cd /opt/apps/esc

# Option A: Use reconfig script (recommended)
./reconfig.sh
# Edit in nano
# Auto-restart

# Option B: Manual
nano .env.docker
# Make changes
./deploy.sh  # Restart
```

### Scenario 5: Disaster Recovery

**Situation**: Server died, need to redeploy on new server

```bash
# 1. Have backups ready
# - .env.docker file
# - Database backup (if external)
# - SSL certificates (if Let's Encrypt)

# 2. On new server
curl -fsSL https://raw.githubusercontent.com/andreas-tuko/esc-compose-prod/main/deploy.sh -o deploy.sh
chmod +x deploy.sh
./deploy.sh

# 3. During installation:
# - Use same domain
# - Use same Docker credentials
# - Choose same SSL option (or restore certs)

# 4. Restore .env.docker
scp old-server:/opt/apps/esc/.env.docker /opt/apps/esc/.env.docker

# 5. Restore database (if needed)
# Follow your database backup restore procedure

# 6. Restart
cd /opt/apps/esc
./deploy.sh
```

### Scenario 6: Rollback to Previous Version

**Situation**: New deployment has issues

```bash
cd /opt/apps/esc

# Option A: Rollback Docker image
docker pull andreastuko/esc:previous-tag
docker tag andreastuko/esc:previous-tag andreastuko/esc:latest
./deploy.sh

# Option B: Rollback code
git log  # Find commit hash
git reset --hard <commit-hash>
./deploy.sh

# Option C: Restore from backup
# Restore .env.docker backup
cp .env.docker.backup.YYYYMMDD_HHMMSS .env.docker
./deploy.sh
```

## Troubleshooting

### Issue: "Configuration file not found"

**Cause**: First-time deployment or config file deleted

**Solution**: This is normal for first deployment. Answer all questions.

### Issue: "Docker login failed"

**Cause**: Incorrect password or network issue

**Solution**:
```bash
# Test Docker login manually
docker login

# If successful, run deploy script again
./deploy.sh
```

### Issue: "Existing config but different domain"

**Situation**: You changed domains outside the script

**Solution**:
```bash
cd /opt/apps/esc
./deploy.sh

# Choose "n" to reconfigure
# Enter new domain
# Update .env.docker with new domain
```

### Issue: "SSL certificate expired"

**Let's Encrypt**:
```bash
# Should auto-renew, but if not:
sudo certbot renew --force-renewal
sudo systemctl reload nginx
```

**Self-signed**:
```bash
# Generate new certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj "/C=KE/ST=Nairobi/L=Nairobi/O=ESC/CN=yourdomain.com"
sudo systemctl reload nginx
```

### Issue: "Want to completely start fresh"

**Solution**:
```bash
cd /opt/apps/esc

# Stop services
./stop.sh

# Remove configuration
rm .deployment_config
rm .env.docker

# Re-run installer
./deploy.sh
# Will be like first-time installation
```

## Best Practices

### Regular Updates

```bash
# Weekly or after each code push
cd /opt/apps/esc
./deploy.sh
# Use existing config
```

### Configuration Changes

```bash
# Use dedicated script for env changes
./reconfig.sh
# Avoids full re-deployment
```

### Before Major Changes

```bash
# Backup current state
cd /opt/apps/esc
cp .env.docker .env.docker.backup.$(date +%Y%m%d)
cp .deployment_config .deployment_config.backup

# If using Let's Encrypt
sudo tar -czf ~/letsencrypt-backup.tar.gz /etc/letsencrypt/

# Database backup (if applicable)
# ... your database backup command ...
```

### Testing Changes

```bash
# Test in staging first
# Use different domain: staging.example.com
# Once verified, deploy to production
```

### Monitoring After Deployment

```bash
# Check logs
cd /opt/apps/esc
./logs.sh web

# Check status
./status.sh

# Monitor for 5-10 minutes
watch -n 5 './status.sh'
```

## Update Checklist

Before re-deploying:

- [ ] Backup .env.docker
- [ ] Backup database (if applicable)
- [ ] Note current version/commit
- [ ] Check for breaking changes in changelog
- [ ] Inform users of maintenance window
- [ ] Have rollback plan ready

During re-deployment:

- [ ] Run ./deploy.sh
- [ ] Use existing config or reconfigure as needed
- [ ] Monitor logs during startup
- [ ] Test critical functionality
- [ ] Check all services healthy

After re-deployment:

- [ ] Verify website accessible
- [ ] Test key features
- [ ] Check logs for errors
- [ ] Monitor resource usage
- [ ] Update documentation if config changed

## Advanced: Automation

### Automated Updates (Cron)

**Not recommended** for production, but possible:

```bash
# Create update script
cat > /opt/apps/esc/auto-update.sh << 'EOF'
#!/bin/bash
cd /opt/apps/esc
git pull
docker pull andreastuko/esc:latest
docker compose -f compose.prod.yaml up -d
EOF

chmod +x /opt/apps/esc/auto-update.sh

# Add to crontab (runs daily at 2 AM)
# Only if you're confident in your CI/CD
(crontab -l; echo "0 2 * * * /opt/apps/esc/auto-update.sh >> /var/log/esc-auto-update.log 2>&1") | crontab -
```

**Better approach**: Use CI/CD with proper testing before auto-deployment.

## Summary

Re-deployment is designed to be:
- ✅ **Smart**: Detects and preserves existing setup
- ✅ **Fast**: Skips unnecessary steps
- ✅ **Safe**: Backs up before changes
- ✅ **Flexible**: Override any setting when needed
- ✅ **Secure**: Never saves passwords

The script remembers your configuration so you can focus on updating your application, not reconfiguring infrastructure.

---

**Questions?** Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) or [README.md](README.md)