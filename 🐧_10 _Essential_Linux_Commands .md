# 🐧 10 Essential Linux Commands for DevOps/SRE (Beyond the Basics)

## 1. `tail -f` / `tail -F` - Real-time Log Monitoring

### Basic Usage
```bash
# Follow log file in real-time
tail -f /var/log/nginx/access.log

# Follow with last 50 lines
tail -n 50 -f /var/log/application.log

# Follow even if file is rotated (capital F)
tail -F /var/log/syslog
```

### Real-World Scenarios
```bash
# Monitor multiple logs simultaneously
tail -f /var/log/nginx/error.log /var/log/php/error.log

# Filter logs in real-time
tail -f /var/log/nginx/access.log | grep "POST /api"

# Monitor Kubernetes pod logs
kubectl logs -f deployment/myapp --tail=100

# Follow with timestamp
tail -f /var/log/app.log | while read line; do echo "$(date): $line"; done
```

**💡 Pro Tip**: Use `less +F /var/log/file.log` for interactive log following with search capabilities (Ctrl+C to stop, F to resume)

---

## 2. `grep` - Pattern Searching

### Basic Usage
```bash
# Case-insensitive search
grep -i "error" /var/log/app.log

# Show line numbers
grep -n "ERROR" application.log

# Search recursively
grep -r "TODO" /var/www/html/
```

### Advanced Real-World Usage
```bash
# Find errors with context (3 lines before, 5 after)
grep -B 3 -A 5 "Exception" app.log

# Count occurrences
grep -c "500" nginx-access.log

# Exclude patterns
grep "ERROR" app.log | grep -v "harmless"

# Multiple patterns (OR)
grep -E "error|warning|critical" syslog

# Show only matching part
grep -o "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}" access.log

# Search for whole words only
grep -w "error" app.log

# Find all files containing pattern
grep -rl "database" /etc/

# Colored output with line numbers
grep --color=always -n "ERROR" app.log | less -R
```

### Real Debugging Scenario
```bash
# Find all 500 errors in last hour with IP addresses
grep "500" /var/log/nginx/access.log | \
  grep "$(date -d '1 hour ago' '+%d/%b/%Y:%H')" | \
  awk '{print $1}' | sort | uniq -c | sort -rn
```

---

## 3. `systemctl` - Service Management (systemd)

### Basic Usage
```bash
# Check service status
systemctl status nginx

# Start/stop/restart service
systemctl start docker
systemctl stop docker
systemctl restart nginx

# Enable/disable on boot
systemctl enable docker
systemctl disable apache2
```

### Real-World Operations
```bash
# Reload configuration without restart
systemctl reload nginx

# Show all failed services
systemctl --failed

# List all active services
systemctl list-units --type=service --state=running

# Check if service is enabled
systemctl is-enabled docker

# Restart service and show logs
systemctl restart myapp && journalctl -u myapp -f

# View service dependencies
systemctl list-dependencies nginx

# Isolate runlevel (switch to rescue mode)
systemctl isolate rescue.target

# Daemon reload (after editing service files)
systemctl daemon-reload
systemctl restart myapp

# Set service to restart on failure
systemctl edit myapp.service
# Add:
[Service]
Restart=on-failure
RestartSec=5s
```

### Custom Service Example
```bash
# Create custom service
cat > /etc/systemd/system/myapp.service <<EOF
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start myapp
systemctl enable myapp
```

---

## 4. `journalctl` - systemd Log Viewer

### Basic Usage
```bash
# View all logs
journalctl

# Follow logs (like tail -f)
journalctl -f

# Logs for specific service
journalctl -u nginx

# Logs since boot
journalctl -b
```

### Advanced Real-World Usage
```bash
# Follow specific service logs
journalctl -u docker.service -f

# Logs from last hour
journalctl --since "1 hour ago"

# Logs from specific time range
journalctl --since "2024-02-19 10:00:00" --until "2024-02-19 11:00:00"

# Show only errors and above
journalctl -p err

# Logs for specific user
journalctl _UID=1000

# Show kernel messages only
journalctl -k

# Reverse order (newest first)
journalctl -r

# Show last 100 lines
journalctl -n 100

# Output in JSON format
journalctl -u myapp -o json

# Check disk usage
journalctl --disk-usage

# Vacuum old logs
journalctl --vacuum-time=7d
journalctl --vacuum-size=500M

# Persistent logging (survives reboots)
mkdir -p /var/log/journal
systemctl restart systemd-journald
```

### Debugging Kubernetes Nodes
```bash
# Check kubelet logs
journalctl -u kubelet -f --since "10 minutes ago"

# Check containerd/docker logs
journalctl -u containerd -f
journalctl -u docker -f

# Check for OOM kills
journalctl -k | grep -i "killed process"
```

---

## 5. `ps aux | grep` - Process Management

### Basic Usage
```bash
# Find specific process
ps aux | grep nginx

# Show process tree
ps auxf

# Show processes by user
ps aux | grep ^ubuntu
```

### Advanced Real-World Usage
```bash
# Find zombie processes
ps aux | grep 'Z'

# Sort by memory usage
ps aux --sort=-%mem | head -20

# Sort by CPU usage
ps aux --sort=-%cpu | head -20

# Show process with full command
ps auxww | grep python

# Find process by port
netstat -tulpn | grep :8080
# Or with ss (modern)
ss -tulpn | grep :8080

# Find process using most memory
ps aux | awk '{print $2, $4, $11}' | sort -k2rn | head -n 10

# Kill all processes matching pattern
pkill -f "celery worker"
# Or
ps aux | grep celery | grep -v grep | awk '{print $2}' | xargs kill -9

# Monitor specific process continuously
watch -n 1 'ps aux | grep nginx'

# Show threads for a process
ps -eLf | grep nginx
```

### Advanced Process Monitoring
```bash
# Use htop (interactive)
htop

# Use top with custom columns
top -o %MEM  # Sort by memory
top -u ubuntu  # Show user's processes

# Find which process is using a file
lsof /var/log/nginx/access.log

# Find all files opened by a process
lsof -p 1234

# Find which process is using a port
lsof -i :3000
```

---

## 6. `df -h` / `du -sh` - Disk Space Management

### Basic Usage
```bash
# Check disk space
df -h

# Check folder size
du -sh /var/log

# Check sizes of all subdirectories
du -sh *
```

### Real-World Disk Management
```bash
# Show disk usage sorted
df -h | sort -k 5 -rn

# Show inode usage (can run out before disk space)
df -i

# Find largest directories
du -h /var | sort -rh | head -20

# Find large files (>100MB)
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Find files modified in last 7 days
find /var/log -type f -mtime -7 -exec du -h {} \; | sort -rh

# Show disk usage excluding specific directories
du -h --exclude=/proc --exclude=/sys / | sort -rh | head -20

# Check specific filesystem type
df -h -t ext4

# Monitor disk usage in real-time
watch -n 5 df -h

# Find what's using deleted space (file deleted but process still has it open)
lsof +L1

# Clean package cache (Ubuntu/Debian)
apt clean
apt autoclean

# Clean journald logs
journalctl --vacuum-size=100M

# Find and delete old log files
find /var/log -name "*.gz" -mtime +30 -delete
find /var/log -name "*.log.*" -mtime +7 -delete
```

### Disk Space Troubleshooting Script
```bash
#!/bin/bash
# Quick disk space diagnostic
echo "=== Disk Usage ==="
df -h

echo -e "\n=== Top 10 Largest Directories in /var ==="
du -h /var 2>/dev/null | sort -rh | head -10

echo -e "\n=== Docker Space Usage ==="
docker system df

echo -e "\n=== Largest Files in /var/log ==="
find /var/log -type f -exec du -h {} \; 2>/dev/null | sort -rh | head -10

echo -e "\n=== Deleted Files Still Open ==="
lsof +L1 2>/dev/null
```

---

## 7. `chmod` / `chown` - Permission Management

### Basic Usage
```bash
# Make script executable
chmod +x deploy.sh

# Change ownership
chown ubuntu:ubuntu file.txt

# Recursive ownership
chown -R www-data:www-data /var/www/html
```

### Real-World Permission Management
```bash
# Common web server permissions
chown -R www-data:www-data /var/www/html
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;

# SSH key permissions (required for security)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 644 ~/.ssh/authorized_keys

# Set SGID on directory (new files inherit group)
chmod g+s /shared/directory

# Set sticky bit (only owner can delete files)
chmod +t /tmp/shared

# Numeric permissions
chmod 755 script.sh    # rwxr-xr-x
chmod 644 config.yml   # rw-r--r--
chmod 600 secret.key   # rw-------
chmod 400 readonly.txt # r--------

# Add execute for user only
chmod u+x script.sh

# Remove write for group and others
chmod go-w file.txt

# Recursive with specific permissions
find /path -type f -exec chmod 644 {} \;
find /path -type d -exec chmod 755 {} \;

# Change ownership of symbolic link (not target)
chown -h user:group symlink

# Show numeric permissions
stat -c '%a %n' file.txt

# ACLs (Access Control Lists)
setfacl -m u:jenkins:rwx /var/www/html
getfacl /var/www/html
```

### Docker Permission Issues
```bash
# Fix Docker socket permissions
sudo chmod 666 /var/run/docker.sock

# Fix ownership after Docker volume mount
docker run -v /host/data:/container/data myimage
chown -R 1000:1000 /host/data  # Match container UID
```

---

## 8. `find` - File Search

### Basic Usage
```bash
# Find by name
find /var/log -name "*.log"

# Find files older than 7 days
find /tmp -mtime +7

# Find and delete
find /var/log -name "*.gz" -delete
```

### Advanced Real-World Usage
```bash
# Find and execute command
find /var/log -name "*.log" -exec ls -lh {} \;

# Find large files (>1GB)
find / -type f -size +1G 2>/dev/null

# Find files by permissions
find /var/www -type f -perm 777

# Find files modified in last 24 hours
find /var/log -type f -mtime -1

# Find empty files/directories
find /tmp -type f -empty
find /tmp -type d -empty

# Find by user
find /home -user ubuntu

# Find SUID files (security audit)
find / -perm -4000 -type f 2>/dev/null

# Find world-writable files (security risk)
find / -type f -perm -002 2>/dev/null

# Find files NOT owned by specific user
find /var/www -not -user www-data

# Find and compress old logs
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;

# Find symlinks
find /usr/bin -type l

# Find by multiple extensions
find . -type f \( -name "*.jpg" -o -name "*.png" \)

# Find recently modified config files
find /etc -name "*.conf" -mtime -1

# Complex: Find, display size, and sort
find /var -type f -size +100M -exec du -h {} \; | sort -rh

# Find files accessed in last 3 days
find /home -atime -3
```

### Cleanup Scripts
```bash
# Delete old Docker images
docker images | grep "months ago" | awk '{print $3}' | xargs docker rmi

# Clean old logs
find /var/log -name "*.log.*" -mtime +30 -delete

# Find and remove node_modules
find . -name "node_modules" -type d -prune -exec rm -rf {} +

# Archive old files
find /backups -name "*.tar.gz" -mtime +90 -exec mv {} /archive/ \;
```

---

## 9. `curl` - HTTP Client

### Basic Usage
```bash
# Simple GET request
curl http://localhost:8080

# View headers
curl -I http://example.com

# Follow redirects
curl -L http://example.com
```

### Advanced Real-World Usage
```bash
# POST JSON data
curl -X POST http://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'

# Authentication
curl -u username:password http://api.example.com/data
curl -H "Authorization: Bearer TOKEN" http://api.example.com/data

# Upload file
curl -F "file=@document.pdf" http://api.example.com/upload

# Download file
curl -O http://example.com/file.tar.gz
curl -o custom-name.tar.gz http://example.com/file.tar.gz

# Resume download
curl -C - -O http://example.com/largefile.iso

# Show progress
curl --progress-bar -O http://example.com/file.zip

# Timeout settings
curl --connect-timeout 10 --max-time 30 http://slow-api.com

# Verbose output (debugging)
curl -v http://api.example.com

# Ignore SSL certificate errors (don't use in production!)
curl -k https://self-signed.com

# Save cookies
curl -c cookies.txt http://example.com/login

# Load cookies
curl -b cookies.txt http://example.com/dashboard

# Custom headers
curl -H "X-Custom-Header: value" http://api.example.com

# Multiple URLs
curl http://site1.com http://site2.com

# Rate limiting test
for i in {1..100}; do curl http://localhost:8080; done

# API health check with retry
curl --retry 5 --retry-delay 2 http://api.example.com/health

# Check if service is up
curl -f http://localhost:8080/health || echo "Service down!"

# Measure response time
curl -w "@curl-format.txt" -o /dev/null -s http://example.com

# curl-format.txt:
time_namelookup:  %{time_namelookup}\n
time_connect:  %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_pretransfer:  %{time_pretransfer}\n
time_redirect:  %{time_redirect}\n
time_starttransfer:  %{time_starttransfer}\n
----------\n
time_total:  %{time_total}\n

# Test load balancer
for i in {1..10}; do curl -s http://lb.example.com | grep -o "server-[0-9]"; done
```

### Kubernetes API Interaction
```bash
# Get pods via API
TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
APISERVER=$(kubectl config view | grep server | cut -f 2- -d ":" | tr -d " ")
curl $APISERVER/api/v1/namespaces/default/pods --header "Authorization: Bearer $TOKEN" --insecure
```

---

## 10. `rsync` - Efficient File Synchronization

### Basic Usage
```bash
# Sync directory
rsync -av /source/ /destination/

# Sync to remote
rsync -avz /local/ user@remote:/path/

# Sync from remote
rsync -avz user@remote:/path/ /local/
```

### Advanced Real-World Usage
```bash
# Full backup with progress
rsync -avzh --progress /data/ /backup/

# Sync with delete (mirror)
rsync -av --delete /source/ /destination/

# Exclude patterns
rsync -av --exclude '*.log' --exclude 'node_modules' /app/ /backup/

# Dry run (test without changes)
rsync -avz --dry-run /source/ /destination/

# Resume interrupted transfer
rsync -avz --partial /large-file user@remote:/path/

# Sync over SSH with custom port
rsync -avz -e "ssh -p 2222" /local/ user@remote:/path/

# Bandwidth limit (KB/s)
rsync -avz --bwlimit=1000 /local/ user@remote:/path/

# Show differences before sync
rsync -avz --itemize-changes /source/ /destination/

# Preserve hard links
rsync -avzH /source/ /destination/

# Sync only newer files
rsync -avu /source/ /destination/

# Remote to remote sync
rsync -avz user@server1:/path/ user@server2:/path/

# Backup with timestamp
rsync -avz /data/ /backup/backup-$(date +%Y%m%d)/

# Sync with rsync daemon
rsync -avz /local/ rsync://remote/module/

# Complex exclude file
cat > rsync-exclude.txt <<EOF
*.tmp
*.log
node_modules/
.git/
EOF

rsync -avz --exclude-from='rsync-exclude.txt' /app/ /backup/
```

### Production Backup Scripts
```bash
#!/bin/bash
# Daily backup script

SOURCE="/var/www/html"
DEST="/backup/website"
DATE=$(date +%Y-%m-%d)
BACKUP_DIR="$DEST/$DATE"

# Create backup
rsync -avz --delete \
  --exclude 'cache/*' \
  --exclude '*.log' \
  --link-dest="$DEST/latest" \
  "$SOURCE/" "$BACKUP_DIR/"

# Update latest symlink
ln -snf "$BACKUP_DIR" "$DEST/latest"

# Clean old backups (keep 7 days)
find "$DEST" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;

# Log completion
echo "Backup completed: $DATE" >> /var/log/backup.log
```

### Docker Volume Backup
```bash
# Backup Docker volume
docker run --rm \
  -v postgres_data:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/postgres-$(date +%Y%m%d).tar.gz -C /data .

# Restore Docker volume
docker run --rm \
  -v postgres_data:/data \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/postgres-20240219.tar.gz -C /data
```

---

## 🎓 Bonus: Command Combinations

### System Monitoring One-liner
```bash
# Show top processes by memory and CPU
watch -n 2 'ps aux --sort=-%mem,-%cpu | head -20'
```

### Log Analysis Pipeline
```bash
# Count error types
cat app.log | grep ERROR | awk '{print $5}' | sort | uniq -c | sort -rn
```

### Network Diagnostic
```bash
# Check if port is open
timeout 2 bash -c "</dev/tcp/localhost/8080" && echo "Port open" || echo "Port closed"
```

### Disk Cleanup Automation
```bash
# One-liner to free up space
sudo journalctl --vacuum-size=100M && \
sudo apt autoclean && \
docker system prune -af --volumes && \
find /var/log -name "*.gz" -mtime +7 -delete
```

---

## 📊 Summary Table

| Command | Primary Use | Best For |
|---------|-------------|----------|
| `tail -f` | Log monitoring | Debugging real-time issues |
| `grep` | Pattern search | Finding errors in logs |
| `systemctl` | Service management | Starting/stopping services |
| `journalctl` | System logs | Debugging systemd services |
| `ps aux` | Process listing | Finding resource hogs |
| `df/du` | Disk usage | Space management |
| `chmod/chown` | Permissions | Security & access control |
| `find` | File search | Cleanup & audits |
| `curl` | HTTP requests | API testing & health checks |
| `rsync` | File sync | Backups & deployments |

---

**🎯 Pro Tip**: Combine these commands with `watch`, `xargs`, and pipes (`|`) for powerful one-liners!
