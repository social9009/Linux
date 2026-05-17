# 🚨 /var Disk Space Crisis - Complete Troubleshooting Guide

## 🔍 The Problem: /var is 90% Full

`/var` contains critical system data:
- **Logs**: `/var/log` (system, application, service logs)
- **Caches**: `/var/cache` (package manager, application caches)
- **Docker**: `/var/lib/docker` (images, containers, volumes)
- **Databases**: `/var/lib/mysql`, `/var/lib/postgresql`
- **Mail**: `/var/mail`, `/var/spool`
- **Temporary**: `/var/tmp`

**⚠️ Critical**: If `/var` fills completely (100%), system services can fail, databases can crash, and logging stops.

---

## 🎯 Step-by-Step Recovery Plan

### Phase 1: Immediate Assessment (2 minutes)

#### Step 1: Check Current Usage
```bash
# Check /var usage
df -h /var

# Output example:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvda1       20G   18G  1.5G  92% /var

# If /var is separate partition
df -h | grep /var

# If it's part of root
df -h /
```

#### Step 2: Identify Top Consumers
```bash
# Quick top 10 directories in /var
sudo du -sh /var/* 2>/dev/null | sort -hr | head -10

# Output example:
# 12G    /var/log
# 4.5G   /var/lib/docker
# 1.2G   /var/cache
# 800M   /var/tmp
# 150M   /var/spool

# More detailed breakdown
sudo du -h /var --max-depth=2 2>/dev/null | sort -hr | head -20
```

#### Step 3: Find Largest Individual Files
```bash
# Find files larger than 100MB in /var
sudo find /var -type f -size +100M -exec ls -lh {} \; 2>/dev/null | \
  awk '{print $5 "\t" $9}' | sort -hr

# Find files larger than 1GB
sudo find /var -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# Top 20 largest files
sudo find /var -type f -exec du -h {} + 2>/dev/null | sort -hr | head -20
```

---

### Phase 2: Common Culprits & Solutions

## 🗂️ Scenario 1: /var/log is the Problem (Most Common)

### Check Log Sizes
```bash
# Detailed log directory analysis
sudo du -sh /var/log/* | sort -hr | head -20

# Find large log files
sudo find /var/log -type f -size +100M -exec ls -lh {} \;

# Check specific log directories
sudo du -sh /var/log/nginx/*
sudo du -sh /var/log/apache2/*
sudo du -sh /var/log/syslog*
sudo du -sh /var/log/journal/*
```

### Solution 1: Clean Old Rotated Logs
```bash
# Remove compressed old logs (older than 7 days)
sudo find /var/log -name "*.gz" -mtime +7 -delete

# Remove numbered logs (.1, .2, .3, etc.)
sudo find /var/log -name "*.log.[0-9]*" -mtime +7 -delete

# Remove old dated logs
sudo find /var/log -name "*.log-*" -mtime +7 -delete

# Check space freed
df -h /var
```

### Solution 2: Truncate Large Active Logs
```bash
# DON'T delete active logs - truncate instead
# Identify large active logs
sudo find /var/log -type f -name "*.log" -size +500M

# Truncate (preserves file handles)
sudo truncate -s 0 /var/log/syslog
sudo truncate -s 0 /var/log/kern.log
sudo truncate -s 0 /var/log/auth.log

# For application logs
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/nginx/error.log

# Alternative: Keep last 10000 lines
sudo tail -10000 /var/log/syslog > /tmp/syslog.tmp
sudo mv /tmp/syslog.tmp /var/log/syslog
```

### Solution 3: Clean journald Logs
```bash
# Check journald usage
sudo journalctl --disk-usage

# Output: Archived and active journals take up 2.1G in the file system.

# Vacuum by size (keep only 200MB)
sudo journalctl --vacuum-size=200M

# Vacuum by time (keep only 7 days)
sudo journalctl --vacuum-time=7d

# Vacuum by files (keep only 5 files)
sudo journalctl --vacuum-files=5

# Verify
sudo journalctl --disk-usage
```

### Solution 4: Fix Log Rotation
```bash
# Force immediate log rotation
sudo logrotate -f /etc/logrotate.conf

# Check logrotate configuration
cat /etc/logrotate.conf

# Configure aggressive rotation for specific logs
sudo nano /etc/logrotate.d/nginx

# Example config:
/var/log/nginx/*.log {
    daily           # Rotate daily instead of weekly
    rotate 7        # Keep only 7 days
    maxsize 100M    # Force rotation if over 100MB
    compress        # Compress rotated logs
    delaycompress   # Compress after 2nd rotation
    notifempty      # Don't rotate if empty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}

# Test configuration
sudo logrotate -d /etc/logrotate.d/nginx
```

---

## 🐳 Scenario 2: Docker Consuming Space

### Check Docker Usage
```bash
# Detailed Docker space usage
docker system df

# Output:
# TYPE            TOTAL    ACTIVE   SIZE      RECLAIMABLE
# Images          25       5        8.5GB     7.2GB (84%)
# Containers      10       2        100MB     80MB (80%)
# Local Volumes   15       3        2.3GB     2.1GB (91%)
# Build Cache     0        0        0B        0B

# Verbose output
docker system df -v
```

### Solution 1: Remove Unused Docker Resources
```bash
# ⚠️ CAUTION: Test in non-production first

# Remove stopped containers only
docker container prune -f

# Remove unused images
docker image prune -a -f

# Remove unused volumes (CAREFUL - data loss possible)
docker volume prune -f

# Remove build cache
docker builder prune -a -f

# Nuclear option: Remove EVERYTHING unused
docker system prune -a --volumes -f

# Check space freed
docker system df
df -h /var
```

### Solution 2: Clean Specific Docker Items
```bash
# List large images
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -hr

# Remove specific unused images
docker rmi $(docker images -f "dangling=true" -q)

# Remove old containers (not running in last 7 days)
docker ps -a --filter "status=exited" --filter "status=created" \
  --format "{{.ID}}\t{{.Status}}" | \
  awk '$2 !~ /ago$/ || $3 > 7 {print $1}' | \
  xargs -r docker rm

# Remove unused volumes manually
docker volume ls -qf dangling=true | xargs -r docker volume rm

# Find large volumes
docker volume ls -q | xargs docker volume inspect | \
  grep -E 'Name|Mountpoint' | paste - - | \
  xargs -I {} sh -c 'echo "{}"; du -sh $(echo "{}" | cut -d'"'"' '"'"' -f4)'
```

### Solution 3: Move Docker Root Directory
```bash
# If /var is on small partition, move Docker to larger one

# Stop Docker
sudo systemctl stop docker

# Edit Docker daemon config
sudo nano /etc/docker/daemon.json

# Add:
{
  "data-root": "/home/docker-data"
}

# Move existing data
sudo rsync -aP /var/lib/docker/ /home/docker-data/

# Restart Docker
sudo systemctl start docker

# Verify
docker info | grep "Docker Root Dir"

# Remove old data after verification
sudo rm -rf /var/lib/docker
```

---

## 📦 Scenario 3: Package Manager Cache

### APT (Debian/Ubuntu)
```bash
# Check cache size
du -sh /var/cache/apt/archives

# Clean package cache
sudo apt clean          # Remove all cached .deb files
sudo apt autoclean      # Remove only outdated .deb files

# Remove old kernels (keep current + 1 previous)
sudo apt autoremove --purge

# Check space freed
df -h /var
```

### YUM/DNF (RHEL/CentOS/Fedora)
```bash
# Check cache size
du -sh /var/cache/yum
du -sh /var/cache/dnf

# Clean all caches
sudo yum clean all      # RHEL/CentOS 7
sudo dnf clean all      # RHEL/CentOS 8+, Fedora

# Remove old kernels
sudo dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)

# Verify
df -h /var
```

---

## 🗄️ Scenario 4: Database Data Growth

### Check Database Sizes
```bash
# MySQL/MariaDB
sudo du -sh /var/lib/mysql/*

# PostgreSQL
sudo du -sh /var/lib/postgresql/*

# MongoDB
sudo du -sh /var/lib/mongodb/*
```

### MySQL Cleanup
```bash
# Connect to MySQL
mysql -u root -p

# Check database sizes
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.TABLES
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC;

# Clean binary logs
SHOW BINARY LOGS;
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);

# Optimize tables
OPTIMIZE TABLE table_name;
```

### PostgreSQL Cleanup
```bash
# Connect as postgres user
sudo -u postgres psql

# Check database sizes
SELECT 
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

# Vacuum databases
VACUUM FULL;
VACUUM ANALYZE;

# Clean old WAL files
SELECT pg_switch_wal();
```

---

## 📧 Scenario 5: Mail Spool

### Check Mail Spool
```bash
# Check size
du -sh /var/mail/*
du -sh /var/spool/mail/*

# List large mailboxes
du -sh /var/mail/* | sort -hr

# Check for bounced emails
du -sh /var/spool/mqueue/*
```

### Clean Mail Spool
```bash
# ⚠️ CAREFUL: This deletes mail

# Remove mail for specific user
sudo rm -rf /var/mail/username

# Clear mail queue
sudo postsuper -d ALL

# Clear deferred queue
sudo postsuper -d ALL deferred
```

---

## 🔄 Scenario 6: Temporary Files

### Clean /var/tmp
```bash
# Check usage
du -sh /var/tmp

# Remove files older than 10 days
sudo find /var/tmp -type f -mtime +10 -delete

# Remove empty directories
sudo find /var/tmp -type d -empty -delete
```

### Clean Core Dumps
```bash
# Find core dumps
sudo find /var -name "core.*" -o -name "core"

# Remove core dumps
sudo find /var -name "core.*" -delete
sudo find /var -name "core" -type f -delete

# Disable core dumps system-wide
echo "* hard core 0" | sudo tee -a /etc/security/limits.conf
```

---

## 🛠️ Advanced Troubleshooting

### Find Deleted But Open Files
```bash
# Files deleted but still held open by processes
sudo lsof +L1 | grep /var

# These files consume space until process is restarted
# Example output:
# nginx     1234 www-data   5u   REG  253,1  2147483648  0 /var/log/nginx/access.log (deleted)

# Restart the process to release space
sudo systemctl restart nginx
```

### Check for Rapidly Growing Files
```bash
# Monitor file size changes in real-time
watch -n 1 'du -sh /var/log/*'

# Find files modified in last 10 minutes
find /var -type f -mmin -10 -exec ls -lh {} \;

# Find which process is writing most
sudo iotop -o
```

### Identify Process Consuming Disk I/O
```bash
# Install iotop
sudo apt install iotop   # Debian/Ubuntu
sudo yum install iotop   # RHEL/CentOS

# Monitor I/O
sudo iotop -o

# Alternative: pidstat
sudo apt install sysstat
sudo pidstat -d 1
```

---

## 🤖 Automation & Prevention

### Automated Cleanup Script
```bash
#!/bin/bash
# /usr/local/bin/var-cleanup.sh

set -e

LOG_FILE="/var/log/var-cleanup.log"
THRESHOLD=80  # Trigger cleanup at 80%

echo "=== Cleanup started at $(date) ===" >> $LOG_FILE

# Check current usage
USAGE=$(df -h /var | tail -1 | awk '{print $5}' | sed 's/%//')

if [ $USAGE -lt $THRESHOLD ]; then
    echo "/var usage is ${USAGE}%, below threshold ${THRESHOLD}%. Skipping." >> $LOG_FILE
    exit 0
fi

echo "/var usage is ${USAGE}%, starting cleanup..." >> $LOG_FILE

# Clean logs
echo "Cleaning old log files..." >> $LOG_FILE
find /var/log -name "*.gz" -mtime +7 -delete
find /var/log -name "*.log.[0-9]*" -mtime +7 -delete
journalctl --vacuum-size=200M

# Clean package cache
echo "Cleaning package cache..." >> $LOG_FILE
if command -v apt &> /dev/null; then
    apt clean
elif command -v yum &> /dev/null; then
    yum clean all
fi

# Clean Docker (if installed)
if command -v docker &> /dev/null; then
    echo "Cleaning Docker resources..." >> $LOG_FILE
    docker system prune -af --filter "until=72h"
fi

# Clean tmp
echo "Cleaning /var/tmp..." >> $LOG_FILE
find /var/tmp -type f -mtime +10 -delete

# Final usage
FINAL_USAGE=$(df -h /var | tail -1 | awk '{print $5}')
echo "Cleanup completed. Final usage: $FINAL_USAGE" >> $LOG_FILE
echo "=== Cleanup ended at $(date) ===" >> $LOG_FILE
echo "" >> $LOG_FILE
```

### Schedule with Cron
```bash
# Make script executable
sudo chmod +x /usr/local/bin/var-cleanup.sh

# Add to crontab (daily at 2 AM)
sudo crontab -e

# Add:
0 2 * * * /usr/local/bin/var-cleanup.sh
```

### systemd Timer (Modern Alternative)
```bash
# Create service file
sudo nano /etc/systemd/system/var-cleanup.service

[Unit]
Description=Clean /var directory
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/var-cleanup.sh

[Install]
WantedBy=multi-user.target

# Create timer file
sudo nano /etc/systemd/system/var-cleanup.timer

[Unit]
Description=Run var cleanup daily
Requires=var-cleanup.service

[Timer]
OnCalendar=daily
OnBootSec=10min
Persistent=true

[Install]
WantedBy=timers.target

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable var-cleanup.timer
sudo systemctl start var-cleanup.timer

# Check status
sudo systemctl list-timers --all
```

---

## 📊 Monitoring & Alerts

### Set Up Prometheus Alert
```yaml
# prometheus-alert.yml
groups:
  - name: disk_space
    interval: 60s
    rules:
      - alert: VarPartitionAlmostFull
        expr: (node_filesystem_avail_bytes{mountpoint="/var"} / node_filesystem_size_bytes{mountpoint="/var"}) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "/var partition is almost full"
          description: "/var has less than 20% free space (current: {{ $value }}%)"
```

### Simple Bash Monitoring Script
```bash
#!/bin/bash
# /usr/local/bin/check-var-usage.sh

THRESHOLD=85
USAGE=$(df -h /var | tail -1 | awk '{print $5}' | sed 's/%//')
HOSTNAME=$(hostname)

if [ $USAGE -ge $THRESHOLD ]; then
    # Send email alert
    echo "WARNING: /var on $HOSTNAME is ${USAGE}% full" | \
        mail -s "[ALERT] /var disk usage on $HOSTNAME" admin@example.com
    
    # Or send to Slack
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"⚠️ /var on $HOSTNAME is ${USAGE}% full\"}" \
        https://hooks.slack.com/services/YOUR/WEBHOOK/URL
fi
```

### Add to Cron (check every hour)
```bash
0 * * * * /usr/local/bin/check-var-usage.sh
```

---

## 🎯 Quick Reference Commands

### Diagnosis
```bash
# Current usage
df -h /var

# Top consumers
sudo du -sh /var/* | sort -hr | head -10

# Largest files
sudo find /var -type f -size +100M -exec ls -lh {} \;

# Deleted but open files
sudo lsof +L1 | grep /var
```

### Quick Cleanup (Safe)
```bash
# Clean logs
sudo find /var/log -name "*.gz" -mtime +7 -delete
sudo journalctl --vacuum-size=200M

# Clean package cache
sudo apt clean || sudo yum clean all

# Clean Docker
docker system prune -af --filter "until=72h"
```

### Emergency Cleanup (When Critical)
```bash
# Truncate largest log file
LARGEST=$(sudo find /var/log -type f -exec du -h {} + | sort -hr | head -1 | awk '{print $2}')
sudo truncate -s 0 "$LARGEST"

# Clean all old logs aggressively
sudo find /var/log -type f -mtime +1 -delete

# Restart services to release deleted files
sudo systemctl restart nginx docker postgresql
```

---

## 📋 Troubleshooting Checklist

```
□ Check current /var usage (df -h /var)
□ Identify top 10 directories (du -sh /var/*)
□ Find largest files (find /var -size +100M)
□ Check for deleted but open files (lsof +L1)
□ Clean old rotated logs (*.gz, *.log.*)
□ Truncate large active logs (truncate -s 0)
□ Clean journald (journalctl --vacuum-size)
□ Clean package cache (apt clean / yum clean)
□ Clean Docker if installed (docker system prune)
□ Check mail spool (/var/mail, /var/spool)
□ Clean /var/tmp (find -mtime +10 -delete)
□ Restart services to release file handles
□ Set up monitoring/alerts
□ Configure automated cleanup
□ Document root cause in runbook
```

---

## 🎓 Interview Answer Template

**When asked in an interview, structure your answer like this:**

1. **Assess**: "First, I'd check `df -h /var` and run `du -sh /var/*` to identify the culprit"

2. **Most Common**: "In my experience, it's usually `/var/log` (80% of cases), `/var/lib/docker` (15%), or package cache (5%)"

3. **Safe Cleanup**: "I'd start with safe operations like cleaning old rotated logs, journald vacuum, and package cache"

4. **Root Cause**: "After freeing space, I'd investigate why it filled up—usually verbose logging or lack of log rotation"

5. **Prevention**: "Then I'd set up proper log rotation, monitoring alerts at 80%, and automated cleanup scripts"

6. **Example**: "I once had a production server where nginx access logs filled /var. I truncated the log, fixed logrotate config, and set up a daily cleanup cron job"

---

**🎯 Bottom Line**: /var filling up is usually logs, Docker, or package cache. Clean safely, investigate root cause, prevent recurrence with automation.
