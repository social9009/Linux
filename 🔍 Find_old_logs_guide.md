# 🔍 Finding & Managing Old Log Files - Complete Guide

## 🎯 The Basic Answer

```bash
find /var/log -type f -mtime +7
```

**What it does**: Lists all files in `/var/log` modified more than 7 days ago

---

## 📚 Understanding `find` Command Deeply

### The `-mtime` Option Explained

**The confusing part**: How `+7`, `-7`, and `7` differ

```bash
-mtime +7   # OLDER than 7 days (modified 8+ days ago)
-mtime -7   # NEWER than 7 days (modified 0-6 days ago)
-mtime 7    # EXACTLY 7 days ago (rare use case)
```

**Visual Timeline**:
```
Today ←─── 7 days ───→ Older
  0   1   2   3   4   5   6   7   8   9  10
  │                       │               │
  └─ -mtime -7 ──────────┘               │
                          └─ +7 starts here
```

**Examples**:
```bash
# Files modified in last week (0-7 days)
find /var/log -type f -mtime -7

# Files modified more than 7 days ago (8+)
find /var/log -type f -mtime +7

# Files modified exactly 7 days ago (rarely useful)
find /var/log -type f -mtime 7
```

---

## 🔧 Practical Variations

### Variation 1: List with Details
```bash
# With file sizes and timestamps
find /var/log -type f -mtime +7 -exec ls -lh {} \;

# Output:
# -rw-r--r-- 1 root root 2.1M Feb 10 10:30 /var/log/syslog.1
# -rw-r--r-- 1 root root 1.5M Feb 09 08:15 /var/log/auth.log.1
```

### Variation 2: With Human-Readable Dates
```bash
# Show files with modification dates
find /var/log -type f -mtime +7 -printf "%TY-%Tm-%Td %TT %p\n" | sort

# Output:
# 2024-02-10 10:30:45 /var/log/syslog.1
# 2024-02-09 08:15:22 /var/log/auth.log.1
```

### Variation 3: Count Only
```bash
# How many files are older than 7 days?
find /var/log -type f -mtime +7 | wc -l

# Output: 42
```

### Variation 4: Total Size
```bash
# Total size of old logs
find /var/log -type f -mtime +7 -exec du -ch {} + | tail -1

# Output: 2.3G total
```

### Variation 5: Specific File Types Only
```bash
# Only .log files
find /var/log -type f -name "*.log" -mtime +7

# Only compressed logs
find /var/log -type f -name "*.gz" -mtime +7

# Only rotated logs (numbered)
find /var/log -type f -name "*.log.[0-9]*" -mtime +7
```

---

## 🛠️ Advanced Use Cases

### Use Case 1: Find Large Old Logs
```bash
# Files older than 7 days AND larger than 100MB
find /var/log -type f -mtime +7 -size +100M

# With details sorted by size
find /var/log -type f -mtime +7 -size +100M -exec ls -lhS {} \;
```

### Use Case 2: Exclude Certain Directories
```bash
# Find old logs but skip /var/log/journal
find /var/log -type f -mtime +7 -path /var/log/journal -prune -o -print

# Skip multiple directories
find /var/log -type f -mtime +7 \
  \( -path /var/log/journal -o -path /var/log/private \) -prune -o -print
```

### Use Case 3: Find by Access Time Instead of Modification
```bash
# Files not accessed in 7 days
find /var/log -type f -atime +7

# Files not changed (metadata) in 7 days
find /var/log -type f -ctime +7
```

**Difference between -mtime, -atime, -ctime**:
```
-mtime  # Modified time (file content changed)
-atime  # Access time (file was read)
-ctime  # Change time (file metadata changed: permissions, owner)
```

### Use Case 4: Multiple Conditions
```bash
# Old logs that are also compressed
find /var/log -type f -mtime +7 -name "*.gz"

# Old AND large logs
find /var/log -type f -mtime +7 -size +50M

# Old OR large logs (either condition)
find /var/log -type f \( -mtime +7 -o -size +100M \)
```

---

## 🧹 Safe Deletion Workflows

### Step 1: Review Before Deleting
```bash
# 1. List what would be deleted
find /var/log -type f -mtime +7

# 2. Save list to file for audit
find /var/log -type f -mtime +7 > /tmp/logs-to-delete.txt

# 3. Review the list
less /tmp/logs-to-delete.txt

# 4. Count how many files
wc -l /tmp/logs-to-delete.txt

# 5. Check total size
find /var/log -type f -mtime +7 -exec du -ch {} + | tail -1
```

### Step 2: Delete Safely
```bash
# Method 1: Using -delete flag (modern)
sudo find /var/log -type f -mtime +7 -delete

# Method 2: Using -exec with rm
sudo find /var/log -type f -mtime +7 -exec rm -f {} \;

# Method 3: Using xargs (faster for many files)
sudo find /var/log -type f -mtime +7 -print0 | xargs -0 rm -f
```

**Important**: Always test with `-print` first before `-delete`!

### Step 3: Delete with Confirmation
```bash
# Interactive deletion (asks for each file)
sudo find /var/log -type f -mtime +7 -exec rm -i {} \;

# Verbose deletion (shows what's being deleted)
sudo find /var/log -type f -mtime +7 -exec rm -v {} \;
```

---

## 🗜️ Compress Instead of Delete

### Compress Old Logs
```bash
# Compress logs older than 7 days
sudo find /var/log -type f -name "*.log" -mtime +7 ! -name "*.gz" -exec gzip {} \;

# Result: file.log becomes file.log.gz (saves ~90% space)
```

### Compress with Logging
```bash
# Log what was compressed
sudo find /var/log -type f -name "*.log" -mtime +7 ! -name "*.gz" \
  -exec gzip {} \; \
  -exec echo "Compressed: {}" \; >> /var/log/compression.log
```

---

## 📊 Comparison Table: Time Options

| Option | Meaning | Example | Finds |
|--------|---------|---------|-------|
| `-mtime +7` | Older than 7 days | Day 8+ | Files from Feb 14 or earlier (if today is Feb 22) |
| `-mtime -7` | Newer than 7 days | Day 0-6 | Files from Feb 15-22 (if today is Feb 22) |
| `-mtime 7` | Exactly 7 days | Day 7 only | Files from Feb 15 (if today is Feb 22) |
| `-mmin +60` | Older than 60 minutes | Hour+ ago | Files modified before 1 hour ago |
| `-mmin -60` | Newer than 60 minutes | Last hour | Files modified in last 60 minutes |

---

## 🎯 Real-World Scenarios

### Scenario 1: Daily Log Cleanup Script
```bash
#!/bin/bash
# /usr/local/bin/cleanup-old-logs.sh

LOG_DIR="/var/log"
DAYS_OLD=7
COMPRESS_DAYS=7
DELETE_DAYS=30
LOGFILE="/var/log/log-cleanup.log"

echo "[$(date)] Starting log cleanup..." >> "$LOGFILE"

# Step 1: Compress logs 7-30 days old
echo "[$(date)] Compressing logs..." >> "$LOGFILE"
find "$LOG_DIR" -type f -name "*.log" \
  -mtime +$COMPRESS_DAYS -mtime -$DELETE_DAYS \
  ! -name "*.gz" \
  -exec gzip {} \; \
  -exec echo "Compressed: {}" >> "$LOGFILE" \;

# Step 2: Delete compressed logs older than 30 days
echo "[$(date)] Deleting old compressed logs..." >> "$LOGFILE"
find "$LOG_DIR" -type f -name "*.gz" \
  -mtime +$DELETE_DAYS \
  -exec rm -f {} \; \
  -exec echo "Deleted: {}" >> "$LOGFILE" \;

echo "[$(date)] Cleanup completed." >> "$LOGFILE"

# Report
COMPRESSED=$(grep -c "Compressed:" "$LOGFILE" | tail -1)
DELETED=$(grep -c "Deleted:" "$LOGFILE" | tail -1)
echo "[$(date)] Summary: $COMPRESSED compressed, $DELETED deleted" >> "$LOGFILE"
```

**Schedule with cron**:
```bash
# Run daily at 2 AM
0 2 * * * /usr/local/bin/cleanup-old-logs.sh
```

---

### Scenario 2: Find Logs from Specific Date Range
```bash
# Logs from last month (30-60 days old)
find /var/log -type f -mtime +30 -mtime -60

# Logs from specific date onwards
find /var/log -type f -newermt "2024-01-01"

# Logs before specific date
find /var/log -type f ! -newermt "2024-01-01"

# Logs in date range
find /var/log -type f -newermt "2024-01-01" ! -newermt "2024-02-01"
```

---

### Scenario 3: Emergency Disk Space Recovery
```bash
# Find and delete largest old logs to free space quickly
sudo find /var/log -type f -mtime +7 -exec du -h {} \; | \
  sort -rh | head -20 | awk '{print $2}' | \
  xargs -I {} sudo rm -f {}

# Or interactively
sudo find /var/log -type f -mtime +7 -exec du -h {} \; | \
  sort -rh | head -20 | \
  while read size file; do
    echo "Delete $file ($size)? [y/N]"
    read answer
    if [[ "$answer" == "y" ]]; then
      sudo rm -f "$file"
      echo "Deleted: $file"
    fi
  done
```

---

### Scenario 4: Archive Old Logs to S3/Remote Storage
```bash
#!/bin/bash
# Archive logs older than 30 days to S3

ARCHIVE_DATE=$(date -d '30 days ago' +%Y%m%d)
BUCKET="s3://my-log-archive"

# Create tarball of old logs
find /var/log -type f -mtime +30 -print0 | \
  tar czf /tmp/logs-${ARCHIVE_DATE}.tar.gz --null -T -

# Upload to S3
aws s3 cp /tmp/logs-${ARCHIVE_DATE}.tar.gz ${BUCKET}/

# Delete local logs after successful upload
if [ $? -eq 0 ]; then
  find /var/log -type f -mtime +30 -delete
  echo "Logs archived and deleted successfully"
fi

# Clean up tarball
rm -f /tmp/logs-${ARCHIVE_DATE}.tar.gz
```

---

## 🧪 Testing Your Command Safely

### Dry Run Checklist
```bash
# 1. Test without permissions (see permission errors)
find /var/log -type f -mtime +7

# 2. Test with sudo (see real results)
sudo find /var/log -type f -mtime +7

# 3. Count files
sudo find /var/log -type f -mtime +7 | wc -l

# 4. Check sizes
sudo find /var/log -type f -mtime +7 -exec du -ch {} + | tail -1

# 5. Sample a few files
sudo find /var/log -type f -mtime +7 | head -10

# 6. Check oldest file
sudo find /var/log -type f -mtime +7 -printf "%T+ %p\n" | sort | head -1

# 7. Check newest file in results
sudo find /var/log -type f -mtime +7 -printf "%T+ %p\n" | sort -r | head -1
```

---

## 🚫 Common Mistakes to Avoid

### Mistake 1: Forgetting -type f
```bash
# Wrong - includes directories
find /var/log -mtime +7

# Right - files only
find /var/log -type f -mtime +7
```

### Mistake 2: Confusing +7 and -7
```bash
# Wrong - finds NEWER files (opposite of what you want)
find /var/log -type f -mtime -7

# Right - finds OLDER files
find /var/log -type f -mtime +7
```

### Mistake 3: Deleting Without Review
```bash
# Dangerous - deletes without checking
sudo find /var/log -type f -mtime +7 -delete

# Safe - review first
sudo find /var/log -type f -mtime +7
# Then if satisfied:
sudo find /var/log -type f -mtime +7 -delete
```

### Mistake 4: Wrong Permissions
```bash
# Fails silently for files you can't read
find /var/log -type f -mtime +7

# Better - use sudo
sudo find /var/log -type f -mtime +7

# Best - redirect errors
find /var/log -type f -mtime +7 2>/dev/null
```

### Mistake 5: Not Accounting for Timezones
```bash
# -mtime uses 24-hour periods from now
# File modified 7 days + 1 hour ago won't match +7

# Use -mmin for minute precision
find /var/log -type f -mmin +10080  # Exactly 7 days = 10080 minutes
```

---

## 📝 Alternative Commands

### Using `ls` with `find`
```bash
# List with timestamps
ls -lt $(find /var/log -type f -mtime +7 2>/dev/null)

# List with human-readable sizes
ls -lhtr $(find /var/log -type f -mtime +7 2>/dev/null)
```

### Using `stat` for Precise Times
```bash
# Find files modified before specific timestamp
find /var/log -type f ! -newermt "2024-02-15 00:00:00"

# Find files accessed before specific date
find /var/log -type f ! -newerat "2024-02-15"
```

---

## 🎓 Interview Answer Template

**When asked in an interview**:

**Basic Answer** (20 seconds):
> "I'd use `find /var/log -type f -mtime +7` to list all files older than 7 days. The `-type f` ensures we only get files, not directories, and `-mtime +7` means modified more than 7 days ago."

**Show Understanding** (add 15 seconds):
> "Before deleting, I'd verify with `find /var/log -type f -mtime +7 -exec du -ch {} + | tail -1` to see total size, then use `-delete` flag or pipe to `xargs rm -f` for actual deletion."

**Show Experience** (add 10 seconds):
> "In production, I'd automate this with a cron job that first compresses logs 7-30 days old with `gzip`, then deletes compressed logs over 30 days old, logging all actions for audit trail."

**Advanced Points** (bonus):
> "I'd also consider using `logrotate` instead of find for a more robust solution, as it handles file rotation, compression, and retention policies in a centralized config file."

---

## 🔧 Quick Reference

```bash
# List old logs
find /var/log -type f -mtime +7

# List with details
find /var/log -type f -mtime +7 -exec ls -lh {} \;

# Count old logs
find /var/log -type f -mtime +7 | wc -l

# Total size
find /var/log -type f -mtime +7 -exec du -ch {} + | tail -1

# Compress old logs
find /var/log -type f -name "*.log" -mtime +7 ! -name "*.gz" -exec gzip {} \;

# Delete old logs (careful!)
sudo find /var/log -type f -mtime +7 -delete

# Specific file types
find /var/log -type f -name "*.log" -mtime +7
find /var/log -type f -name "*.gz" -mtime +30
```

---

**🎯 Bottom Line**: `find` with `-mtime +7` is the standard way to find old logs. Always test first, compress before deleting, and automate with cron for ongoing maintenance.
