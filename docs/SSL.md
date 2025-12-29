# SSL Certificate Configuration Guide

Complete guide for SSL/TLS certificate setup with your ESC Django deployment.

## Table of Contents

- [Overview](#overview)
- [SSL Options](#ssl-options)
- [Let's Encrypt Setup](#lets-encrypt-setup)
- [Self-Signed Certificates](#self-signed-certificates)
- [Cloudflare SSL](#cloudflare-ssl)
- [Certificate Management](#certificate-management)
- [Troubleshooting](#troubleshooting)

## Overview

The deployment script offers three SSL options:

1. **Let's Encrypt** - Free, trusted, auto-renewing certificates (requires valid domain)
2. **Self-Signed** - Works with IP addresses, shows browser warnings
3. **None** - HTTP only, rely on Cloudflare for SSL

## SSL Options

### Option 1: Let's Encrypt (Recommended for Domains)

**Best for:**
- Production deployments
- Valid domain names
- Public-facing websites
- No browser warnings

**Requirements:**
- Valid domain name
- Domain DNS pointing to server
- Port 80/443 accessible
- Valid email address

**Features:**
- ✅ Free forever
- ✅ Trusted by all browsers
- ✅ Auto-renewal (90-day certificates)
- ✅ Professional appearance
- ✅ Full HTTPS support

### Option 2: Self-Signed Certificate

**Best for:**
- Development/testing
- IP-based access
- Internal networks
- When domain not available

**Requirements:**
- None (works immediately)

**Features:**
- ✅ Works with IP addresses
- ✅ Immediate setup
- ✅ Full encryption
- ⚠️ Browser security warnings
- ⚠️ Not trusted by default

### Option 3: No SSL (Cloudflare Only)

**Best for:**
- Using Cloudflare proxy
- Quick setup
- When SSL handled externally

**Features:**
- ✅ Fastest setup
- ✅ Cloudflare handles SSL
- ✅ No server configuration
- ⚠️ HTTP to server (Cloudflare encrypts client-side)

## Let's Encrypt Setup

### During Installation

When running the deployment script:

```bash
./deploy.sh

# Choose SSL option
SSL Certificate Configuration
Choose SSL certificate option:
  1) Let's Encrypt (Free, auto-renewing, requires valid domain)
  2) Self-signed (Works with IP address, not trusted by browsers)
  3) None (Use Cloudflare SSL only)
Select option [1/2/3]: 1

# Enter email for notifications
Email for Let's Encrypt notifications: admin@yourdomain.com
```

### What Happens

1. **Certbot Installation** - Installs Let's Encrypt client
2. **Certificate Request** - Requests cert for your domain and www subdomain
3. **Nginx Configuration** - Configures HTTPS with proper settings
4. **Auto-Renewal Setup** - Adds daily cron job for renewal

### After Installation

Your site will be accessible via:
- `https://yourdomain.com` ✅
- `https://www.yourdomain.com` ✅
- `http://yourdomain.com` → redirects to HTTPS

### Certificate Details

- **Location**: `/etc/letsencrypt/live/yourdomain.com/`
- **Files**:
  - `fullchain.pem` - Full certificate chain
  - `privkey.pem` - Private key
  - `chain.pem` - Intermediate certificates
  - `cert.pem` - Certificate only
- **Validity**: 90 days
- **Renewal**: Automatic (daily check at 3 AM)

### Manual Certificate Operations

```bash
# Check certificate status
sudo certbot certificates

# Manual renewal (for testing)
sudo certbot renew --dry-run

# Force renewal
sudo certbot renew --force-renewal

# Revoke certificate
sudo certbot revoke --cert-path /etc/letsencrypt/live/yourdomain.com/cert.pem
```

### Cloudflare Integration with Let's Encrypt

If using Cloudflare with Let's Encrypt:

1. **Cloudflare SSL Mode**: Set to **Full (strict)**
   - Dashboard → SSL/TLS → Overview
   - Select "Full (strict)"

2. **Why Full (strict)?**
   - Cloudflare validates your server's certificate
   - End-to-end encryption
   - Maximum security

3. **DNS Configuration**:
   ```
   Type: A
   Name: @ (or your subdomain)
   Content: YOUR_SERVER_IP
   Proxy: Enabled (orange cloud)
   ```

## Self-Signed Certificates

### During Installation

```bash
./deploy.sh

# Choose self-signed option
Select option [1/2/3]: 2

⚠ Self-signed certificates will show security warnings in browsers
ℹ This is useful for testing or IP-based access
```

### What Happens

1. **Certificate Generation** - Creates 2048-bit RSA certificate
2. **DH Parameters** - Generates Diffie-Hellman parameters for security
3. **Nginx Configuration** - Configures HTTPS with self-signed cert
4. **Validity**: 365 days (1 year)

### Certificate Details

- **Location**: `/etc/nginx/ssl/`
- **Files**:
  - `selfsigned.crt` - Certificate
  - `selfsigned.key` - Private key
  - `dhparam.pem` - DH parameters

### Accessing with Self-Signed Certificate

**Via Domain:**
```
https://yourdomain.com
```

**Via IP Address:**
```
https://YOUR_SERVER_IP
```

**Browser Warning:**
You'll see warnings like:
- "Your connection is not private"
- "NET::ERR_CERT_AUTHORITY_INVALID"
- "This site is not secure"

**To Proceed:**
- Chrome: Click "Advanced" → "Proceed to [site]"
- Firefox: Click "Advanced" → "Accept the Risk and Continue"
- Safari: Click "Show Details" → "visit this website"

### Trust Self-Signed Certificate (Optional)

**On Your Computer (Development):**

#### Windows
1. Download the certificate: `scp user@server:/etc/nginx/ssl/selfsigned.crt .`
2. Double-click `selfsigned.crt`
3. Click "Install Certificate"
4. Select "Local Machine"
5. Select "Place certificates in the following store"
6. Choose "Trusted Root Certification Authorities"
7. Finish and restart browser

#### macOS
```bash
# Download certificate
scp user@server:/etc/nginx/ssl/selfsigned.crt .

# Trust certificate
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain selfsigned.crt

# Restart browser
```

#### Linux
```bash
# Download certificate
scp user@server:/etc/nginx/ssl/selfsigned.crt .

# Install certificate
sudo cp selfsigned.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Restart browser
```

### Renew Self-Signed Certificate

Self-signed certificates expire after 1 year. To renew:

```bash
# Backup old certificate
sudo cp /etc/nginx/ssl/selfsigned.crt /etc/nginx/ssl/selfsigned.crt.old
sudo cp /etc/nginx/ssl/selfsigned.key /etc/nginx/ssl/selfsigned.key.old

# Generate new certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj "/C=KE/ST=Nairobi/L=Nairobi/O=ESC/CN=yourdomain.com"

# Reload Nginx
sudo systemctl reload nginx
```

### Cloudflare with Self-Signed Certificate

**Not Recommended** - Cloudflare's "Full (strict)" mode won't work with self-signed certificates.

**Options:**
1. **Use "Full" mode** (not strict) - Less secure, validates encryption but not certificate
2. **Upload certificate to Cloudflare** - Custom SSL option (paid plans)
3. **Use Let's Encrypt instead** - Best option

## Cloudflare SSL

### Option 1: Flexible SSL (No Server Certificate)

**Setup during installation:**
```bash
Select option [1/2/3]: 3  # None
```

**Cloudflare Configuration:**
1. SSL/TLS → Overview
2. Select **Flexible**
3. Enable "Always Use HTTPS"

**How it works:**
```
Browser → HTTPS → Cloudflare → HTTP → Your Server
```

**Pros:**
- ✅ Quick setup
- ✅ Browser sees HTTPS
- ✅ Free

**Cons:**
- ⚠️ Not end-to-end encrypted
- ⚠️ HTTP between Cloudflare and server

### Option 2: Full SSL (with Self-Signed)

**Setup during installation:**
```bash
Select option [1/2/3]: 2  # Self-signed
```

**Cloudflare Configuration:**
1. SSL/TLS → Overview
2. Select **Full** (not strict)

**How it works:**
```
Browser → HTTPS → Cloudflare → HTTPS (self-signed) → Your Server
```

**Pros:**
- ✅ End-to-end encryption
- ✅ No browser warnings (Cloudflare validates)

### Option 3: Full (Strict) with Let's Encrypt

**Setup during installation:**
```bash
Select option [1/2/3]: 1  # Let's Encrypt
```

**Cloudflare Configuration:**
1. SSL/TLS → Overview
2. Select **Full (strict)**

**How it works:**
```
Browser → HTTPS → Cloudflare → HTTPS (Let's Encrypt) → Your Server
```

**Pros:**
- ✅ Maximum security
- ✅ Trusted certificates
- ✅ End-to-end encryption
- ✅ Best practice

## Certificate Management

### Check Certificate Expiry

**Let's Encrypt:**
```bash
sudo certbot certificates
```

**Self-Signed:**
```bash
openssl x509 -in /etc/nginx/ssl/selfsigned.crt -noout -dates
```

### View Certificate Details

```bash
# Let's Encrypt
openssl x509 -in /etc/letsencrypt/live/yourdomain.com/cert.pem -text -noout

# Self-Signed
openssl x509 -in /etc/nginx/ssl/selfsigned.crt -text -noout
```

### Test SSL Configuration

**Online Tools:**
- https://www.ssllabs.com/ssltest/ - Comprehensive SSL test
- https://www.digicert.com/help/ - Quick certificate checker

**Command Line:**
```bash
# Test SSL connection
openssl s_client -connect yourdomain.com:443

# Check certificate chain
openssl s_client -showcerts -connect yourdomain.com:443

# Verify certificate
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Backup Certificates

**Let's Encrypt:**
```bash
# Backup entire Let's Encrypt directory
sudo tar -czf letsencrypt-backup-$(date +%Y%m%d).tar.gz /etc/letsencrypt/

# Store securely
```

**Self-Signed:**
```bash
# Backup certificates
sudo tar -czf ssl-backup-$(date +%Y%m%d).tar.gz /etc/nginx/ssl/
```

### Restore Certificates

**Let's Encrypt:**
```bash
# Stop nginx
sudo systemctl stop nginx

# Restore
sudo tar -xzf letsencrypt-backup-YYYYMMDD.tar.gz -C /

# Start nginx
sudo systemctl start nginx
```

**Self-Signed:**
```bash
# Restore
sudo tar -xzf ssl-backup-YYYYMMDD.tar.gz -C /

# Reload nginx
sudo systemctl reload nginx
```

## Troubleshooting

### Let's Encrypt Issues

#### "Failed to obtain certificate"

**Causes:**
1. Domain not pointing to server
2. Port 80/443 blocked
3. Firewall blocking traffic

**Solutions:**
```bash
# Check DNS
nslookup yourdomain.com
# Should return your server IP

# Check ports
sudo netstat -tlnp | grep -E ':80|:443'

# Check firewall
sudo ufw status

# Allow ports
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

#### "Rate limit exceeded"

Let's Encrypt has rate limits:
- 50 certificates per domain per week
- 5 failures per hour

**Solution:**
```bash
# Wait 1 hour for failure limit reset
# Or use staging for testing:
sudo certbot certonly --staging --standalone -d yourdomain.com
```

#### "Certificate about to expire" emails

**Normal** - Renewal should happen automatically.

**To check:**
```bash
# View renewal configuration
sudo cat /etc/letsencrypt/renewal/yourdomain.com.conf

# Test renewal
sudo certbot renew --dry-run

# Check cron
sudo crontab -l | grep certbot
```

### Self-Signed Issues

#### "Certificate has expired"

```bash
# Generate new certificate (see "Renew Self-Signed Certificate" section above)
```

#### "Browser still shows warning after trusting certificate"

**Solutions:**
1. Clear browser cache
2. Restart browser completely
3. Check certificate was imported to correct store
4. Try different browser to verify

### Nginx Issues

#### "Nginx won't start after SSL setup"

```bash
# Check configuration
sudo nginx -t

# Check certificate files exist
ls -la /etc/letsencrypt/live/yourdomain.com/  # Let's Encrypt
ls -la /etc/nginx/ssl/  # Self-signed

# Check certificate permissions
sudo chmod 644 /etc/nginx/ssl/selfsigned.crt  # If self-signed
sudo chmod 600 /etc/nginx/ssl/selfsigned.key
```

#### "Mixed content warnings"

Occurs when HTTPS page loads HTTP resources.

**Fix in Django:**
```python
# settings.py
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

### General SSL Issues

#### "This site can't provide a secure connection"

**Check:**
```bash
# Verify Nginx is listening on 443
sudo netstat -tlnp | grep :443

# Check Nginx error logs
sudo tail -f /var/log/nginx/esc_error.log

# Restart Nginx
sudo systemctl restart nginx
```

#### "ERR_SSL_PROTOCOL_ERROR"

**Causes:**
1. Nginx not configured for SSL
2. Wrong certificate path
3. Certificate/key mismatch

**Check:**
```bash
# Verify SSL configuration
sudo nginx -t

# Check certificate matches key (outputs should match)
openssl x509 -noout -modulus -in /path/to/cert.crt | openssl md5
openssl rsa -noout -modulus -in /path/to/key.key | openssl md5
```

## Best Practices

### Production Recommendations

1. **Use Let's Encrypt** for domains
2. **Enable HSTS** (already configured)
3. **Keep Nginx updated**: `sudo apt update && sudo apt upgrade nginx`
4. **Monitor certificate expiry**
5. **Use Cloudflare Full (strict)** mode
6. **Regular backups** of certificates

### Security Checklist

- [ ] SSL/TLS configured (not just Cloudflare)
- [ ] HSTS enabled
- [ ] Certificate auto-renewal working
- [ ] Strong ciphers configured (TLS 1.2+)
- [ ] HTTP redirects to HTTPS
- [ ] Certificate backups stored securely
- [ ] Monitoring for certificate expiry

### Testing Checklist

After SSL setup:

- [ ] Access via HTTPS works
- [ ] HTTP redirects to HTTPS
- [ ] No browser warnings (Let's Encrypt)
- [ ] SSL Labs test shows A or A+ rating
- [ ] Cloudflare SSL mode correct
- [ ] Auto-renewal configured (Let's Encrypt)

## Additional Resources

- **Let's Encrypt**: https://letsencrypt.org/docs/
- **Certbot**: https://certbot.eff.org/
- **SSL Labs Test**: https://www.ssllabs.com/ssltest/
- **Mozilla SSL Configuration**: https://ssl-config.mozilla.org/
- **Nginx SSL Guide**: https://nginx.org/en/docs/http/configuring_https_servers.html

---

**Need Help?** Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) or open an issue on GitHub.