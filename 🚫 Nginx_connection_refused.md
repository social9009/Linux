# 🚫 NGINX "Connection Refused" - Complete Troubleshooting Guide

## 🎯 Understanding "Connection Refused"

**What it means**: The TCP connection couldn't be established at all

**Key difference**:
```
Connection Refused  → Port not listening (TCP handshake failed)
Connection Timeout  → Firewall blocking or host unreachable
502 Bad Gateway     → NGINX running but backend down
504 Gateway Timeout → Backend too slow to respond
```

**Visual**:
```
Browser → NGINX → Backend App
   ❌ Refused here = NGINX not listening
         ❌ Refused here = Backend not listening
```

---

## 🔍 Systematic Troubleshooting (8 Steps)

### Step 1: Reproduce & Verify (30 seconds)

#### Test from Command Line
```bash
# Test HTTP
curl -v http://localhost

# Output if refused:
# curl: (7) Failed to connect to localhost port 80: Connection refused

# Test HTTPS
curl -v https://localhost

# Test specific port
curl -v http://localhost:8080

# Test from external IP
curl -v http://your-server-ip
```

#### What Each Error Means
```bash
# Connection refused
curl: (7) Failed to connect to localhost port 80: Connection refused
→ Nothing listening on that port

# Connection timeout
curl: (28) Failed to connect to example.com port 80: Connection timed out
→ Firewall blocking or host down

# Could not resolve host
curl: (6) Could not resolve host: example.com
→ DNS issue, not NGINX

# Empty reply
curl: (52) Empty reply from server
→ Service crashed during connection
```

---

### Step 2: Check NGINX Service Status (30 seconds)

```bash
# Check if NGINX is running
sudo systemctl status nginx

# Look for these indicators:
# Active: active (running) ✅
# Active: inactive (dead)  ❌
# Active: failed           ❌

# If failed, check why
sudo journalctl -u nginx -n 50 --no-pager

# Recent errors only
sudo journalctl -u nginx -p err -n 20
```

#### Common Status Issues

**Issue 1: NGINX Failed to Start**
```bash
# Output:
# nginx.service - A high performance web server
# Loaded: loaded (/lib/systemd/system/nginx.service)
# Active: failed (Result: exit-code)
# Process: 1234 ExecStart=/usr/sbin/nginx (code=exited, status=1/FAILURE)

# Check what went wrong
sudo nginx -t

# Common errors:
# nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
# nginx: [emerg] invalid host in upstream "backend" in /etc/nginx/sites-enabled/default:10
# nginx: [emerg] "server" directive is not allowed here in /etc/nginx/nginx.conf:5
```

**Issue 2: NGINX Stopped Unexpectedly**
```bash
# Check for crashes
sudo journalctl -u nginx | grep -E "killed|segfault|core"

# Check system messages
dmesg | tail -50

# Out of memory killer?
grep -i "killed process" /var/log/syslog
```

---

### Step 3: Check Which Ports Are Listening (1 minute)

```bash
# Method 1: netstat (older)
sudo netstat -tulnp | grep nginx

# Output if running correctly:
# tcp  0  0  0.0.0.0:80    0.0.0.0:*  LISTEN  1234/nginx: master
# tcp6 0  0  :::80         :::*       LISTEN  1234/nginx: master

# Method 2: ss (modern, faster)
sudo ss -tulnp | grep nginx

# Method 3: lsof
sudo lsof -i :80

# Check all listening ports
sudo ss -tulnp | grep LISTEN
```

#### Interpreting Results

**Scenario A: NGINX Listening on Port 80** ✅
```bash
tcp  0  0  0.0.0.0:80    0.0.0.0:*  LISTEN  1234/nginx
→ NGINX is working, problem is elsewhere (firewall, DNS, etc.)
```

**Scenario B: Different Process on Port 80** ❌
```bash
tcp  0  0  0.0.0.0:80    0.0.0.0:*  LISTEN  5678/apache2
→ Apache (or other) is using port 80 instead of NGINX
```

**Scenario C: Nothing on Port 80** ❌
```bash
(no output)
→ Nothing listening, NGINX is not running or misconfigured
```

**Scenario D: Listening on Wrong Interface** ⚠️
```bash
tcp  0  0  127.0.0.1:80  0.0.0.0:*  LISTEN  1234/nginx
→ Only listening on localhost, not externally accessible
```

---

### Step 4: Verify NGINX Configuration (2 minutes)

#### Test Configuration Syntax
```bash
# Test config without applying
sudo nginx -t

# Output if successful:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Output if error:
# nginx: [emerg] unexpected "}" in /etc/nginx/sites-enabled/myapp:15
# nginx: configuration file /etc/nginx/nginx.conf test failed
```

#### Check Active Configuration
```bash
# View main config
cat /etc/nginx/nginx.conf

# Check what's enabled
ls -la /etc/nginx/sites-enabled/

# View your site config
cat /etc/nginx/sites-enabled/default
# or
cat /etc/nginx/sites-enabled/myapp.conf
```

#### Common Configuration Issues

**Issue 1: Port Not Specified**
```nginx
# Wrong
server {
    server_name example.com;
    # Missing listen directive!
}

# Correct
server {
    listen 80;
    listen [::]:80;  # IPv6
    server_name example.com;
}
```

**Issue 2: Listening on Wrong Port**
```nginx
# Check what port you're actually listening on
server {
    listen 8080;  # ← Is this what you intended?
    server_name example.com;
}
```

**Issue 3: Binding to Localhost Only**
```nginx
# Wrong - only accessible from server itself
server {
    listen 127.0.0.1:80;
    server_name example.com;
}

# Correct - accessible from anywhere
server {
    listen 80;
    server_name example.com;
}
```

---

### Step 5: Check Backend Application (2 minutes)

If NGINX is running but shows connection refused, the backend might be down.

#### Identify Backend Configuration
```bash
# Find proxy_pass directive
grep -r "proxy_pass" /etc/nginx/sites-enabled/

# Output example:
# proxy_pass http://localhost:3000;
# proxy_pass http://127.0.0.1:8080;
# proxy_pass http://unix:/run/app.sock;
```

#### Test Backend Directly

**If using TCP port**:
```bash
# Check if backend is listening
sudo ss -tulnp | grep :3000

# Test directly
curl http://localhost:3000

# Check backend process
ps aux | grep node  # or python, java, etc.
```

**If using Unix socket**:
```bash
# Check if socket exists
ls -la /run/app.sock

# Test with curl
curl --unix-socket /run/app.sock http://localhost/

# Check socket permissions
stat /run/app.sock
# Should be: srwxrwxrwx (socket, readable by nginx)
```

#### Common Backend Issues

**Issue 1: Backend Not Running**
```bash
# Start your application
sudo systemctl start myapp
sudo systemctl status myapp

# Or manually
cd /opt/myapp
node server.js &
```

**Issue 2: Backend on Wrong Port**
```bash
# NGINX expects port 3000:
# proxy_pass http://localhost:3000;

# But app actually running on 8080:
$ sudo ss -tulnp | grep node
tcp  0  0  0.0.0.0:8080  0.0.0.0:*  LISTEN  5678/node

# Fix: Update NGINX config or app config
```

**Issue 3: Backend Crashed**
```bash
# Check application logs
sudo journalctl -u myapp -n 100

# Check for crashes
tail -f /var/log/myapp/error.log

# Look for OOM kills
dmesg | grep -i "killed process"
```

---

### Step 6: Check Firewall Rules (1 minute)

#### UFW (Ubuntu Firewall)
```bash
# Check status
sudo ufw status verbose

# Output:
# Status: active
# To                         Action      From
# --                         ------      ----
# 80/tcp                     ALLOW       Anywhere

# If port 80 not listed, add it:
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

#### iptables
```bash
# Check rules
sudo iptables -L -n -v

# Check if port 80 is blocked
sudo iptables -L INPUT -n -v | grep 80

# Allow port 80 (if needed)
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT

# Save rules
sudo netfilter-persistent save  # Debian/Ubuntu
sudo service iptables save      # RHEL/CentOS
```

#### firewalld (RHEL/CentOS)
```bash
# Check status
sudo firewall-cmd --state

# List allowed services
sudo firewall-cmd --list-all

# Allow HTTP
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

### Step 7: Check Cloud/Network Security (Cloud environments)

#### AWS Security Groups
```bash
# Via AWS CLI
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# Check for inbound rule:
# Protocol: TCP
# Port: 80
# Source: 0.0.0.0/0 (or your IP)

# Add rule if missing:
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxxx \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

#### GCP Firewall Rules
```bash
# List firewall rules
gcloud compute firewall-rules list

# Create rule allowing HTTP
gcloud compute firewall-rules create allow-http \
    --allow tcp:80 \
    --source-ranges 0.0.0.0/0 \
    --target-tags http-server
```

#### Azure Network Security Groups
```bash
# List NSG rules
az network nsg rule list --resource-group myResourceGroup --nsg-name myNSG

# Create rule
az network nsg rule create \
    --resource-group myResourceGroup \
    --nsg-name myNSG \
    --name allow-http \
    --protocol tcp \
    --priority 100 \
    --destination-port-range 80 \
    --access allow
```

---

### Step 8: Check SELinux/AppArmor (Security modules)

#### SELinux
```bash
# Check if enabled
getenforce
# Output: Enforcing / Permissive / Disabled

# If Enforcing, check denials
sudo ausearch -m avc -ts recent

# Allow NGINX to connect to network
sudo setsebool -P httpd_can_network_connect 1

# Temporarily disable (testing only!)
sudo setenforce 0

# Re-enable
sudo setenforce 1
```

#### AppArmor
```bash
# Check status
sudo aa-status

# Check NGINX profile
sudo aa-status | grep nginx

# Temporarily disable (testing)
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# Re-enable
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

---

## 🛠️ Real-World Scenarios & Solutions

### Scenario 1: Port 80 Already in Use

**Symptoms**: NGINX won't start, says "Address already in use"

**Diagnosis**:
```bash
# Find what's using port 80
sudo lsof -i :80

# Output:
# COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
# apache2  1234     root    4u  IPv6  12345      0t0  TCP *:http (LISTEN)
```

**Solutions**:
```bash
# Option A: Stop the conflicting service
sudo systemctl stop apache2
sudo systemctl start nginx

# Option B: Change NGINX to different port
# Edit /etc/nginx/sites-enabled/default
server {
    listen 8080;  # Instead of 80
}
sudo nginx -t
sudo systemctl restart nginx

# Option C: Make them coexist (virtual hosts)
# Configure Apache on 8080, NGINX on 80
# NGINX proxies to Apache when needed
```

---

### Scenario 2: NGINX Running But Backend App Down

**Symptoms**: NGINX service is active, but curl shows connection refused

**Diagnosis**:
```bash
# NGINX is running
$ sudo systemctl status nginx
Active: active (running)

# But backend isn't
$ curl http://localhost:3000
curl: (7) Failed to connect to localhost port 3000: Connection refused

# Check backend
$ sudo ss -tulnp | grep :3000
(no output)
```

**Solution**:
```bash
# Start backend application
sudo systemctl start myapp

# Or manually
cd /opt/myapp
npm start &

# Verify it's listening
sudo ss -tulnp | grep :3000

# Test through NGINX
curl http://localhost
```

---

### Scenario 3: Working Locally But Not Externally

**Symptoms**: `curl http://localhost` works, but `curl http://public-ip` fails

**Diagnosis**:
```bash
# Works locally
$ curl http://localhost
OK

# Fails externally
$ curl http://54.123.45.67
curl: (7) Failed to connect to 54.123.45.67 port 80: Connection refused
```

**Check binding**:
```bash
# Is NGINX listening on 0.0.0.0 or 127.0.0.1?
$ sudo ss -tulnp | grep nginx
tcp  0  0  127.0.0.1:80  0.0.0.0:*  LISTEN  1234/nginx
         ^^^^^^^^^^^ Only localhost!
```

**Solution**:
```nginx
# Edit /etc/nginx/sites-enabled/default
server {
    # Wrong - localhost only
    listen 127.0.0.1:80;
    
    # Correct - all interfaces
    listen 80;
    listen [::]:80;
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### Scenario 4: Works After Restart, Fails After Time

**Symptoms**: NGINX works after restart, but fails hours later

**Diagnosis**:
```bash
# Check for crashes
sudo journalctl -u nginx --since "1 hour ago" | grep -E "killed|signal|crash"

# Check system resources
free -h
df -h

# Check for OOM kills
sudo dmesg | grep -i "killed process" | grep nginx
```

**Common Causes**:
1. Out of memory (OOM killer)
2. File descriptor limit
3. Worker process crash
4. Log file filling disk

**Solutions**:
```bash
# Increase memory limits
# Edit /etc/systemd/system/nginx.service.d/override.conf
[Service]
MemoryLimit=2G

# Increase file descriptors
# Edit /etc/nginx/nginx.conf
worker_rlimit_nofile 65535;

# Reload systemd
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

---

## 🔧 Quick Fix Checklist

```bash
# 1. Is NGINX running?
sudo systemctl status nginx

# 2. Is it listening on the right port?
sudo ss -tulnp | grep nginx

# 3. Config valid?
sudo nginx -t

# 4. Backend running?
sudo ss -tulnp | grep :3000  # or your port

# 5. Firewall allowing traffic?
sudo ufw status

# 6. Check NGINX logs
sudo tail -f /var/log/nginx/error.log

# 7. Test backend directly
curl http://localhost:3000

# 8. Restart everything
sudo systemctl restart nginx
sudo systemctl restart myapp
```

---

## 📊 Diagnostic Commands Summary

| Purpose | Command | What to Look For |
|---------|---------|------------------|
| Service status | `systemctl status nginx` | Active/Failed |
| Listening ports | `ss -tulnp \| grep nginx` | Port 80/443 |
| Config test | `nginx -t` | Syntax errors |
| Logs | `journalctl -u nginx -n 50` | Error messages |
| Process list | `ps aux \| grep nginx` | Running processes |
| Firewall | `ufw status` | Port 80/443 allowed |
| Backend check | `ss -tulnp \| grep :3000` | App listening |
| Test connection | `curl -v http://localhost` | Connection result |

---

## 🎓 Interview Answer Template

**Structure for a perfect answer**:

**1. Reproduce** (10 seconds):
> "First, I'd reproduce the error with `curl -v http://localhost` to confirm it's truly connection refused and not a different error like 502 or timeout."

**2. Check NGINX** (15 seconds):
> "Then check if NGINX is running with `systemctl status nginx` and listening on port 80 with `ss -tulnp | grep nginx`. If not listening, I'd test the config with `nginx -t` and check logs with `journalctl -u nginx`."

**3. Check Backend** (15 seconds):
> "If NGINX is fine, I'd verify the backend app is running and listening on the port specified in the `proxy_pass` directive. Test with `ss -tulnp | grep :3000` and `curl http://localhost:3000`."

**4. Network Layer** (10 seconds):
> "Next, check firewall rules with `ufw status` or `iptables -L`, and on cloud instances, verify security groups allow inbound traffic on port 80."

**5. Fix & Verify** (10 seconds):
> "After identifying the issue, restart the failed service, update config if needed, and verify with `curl http://localhost`. Finally, test from external IP to ensure it's accessible."

**6. Real Example** (bonus):
> "I once debugged this where NGINX was bound to `127.0.0.1:80` instead of `0.0.0.0:80`, so it worked locally but not externally. Changed the listen directive and restarted NGINX."

---

**🎯 Bottom Line**: Connection refused = something not listening. Check NGINX → Backend → Firewall in that order.
