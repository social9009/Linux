# 📝 Log Rotation Script - Complete Explanation

## 🎯 What This Script Does

**Goal**: Automatically manage log files to save disk space
- **Compress** old logs (older than 7 days) → makes them smaller
- **Delete** very old logs (older than 30 days) → frees up space
- **Log** all actions → know what happened

**Why?** Logs grow constantly. Without management, they fill up your disk!

---

## 📊 Visual Timeline

```
Today ←─── 7 days ───┼─── 30 days ───→ Old
  │                   │                  │
  │                   │                  │
  ▼                   ▼                  ▼
[Keep as-is]    [Compress it]      [Delete it]
 app.log         app.log.gz        [removed]
```

**Example**:
- File from **today** → Keep it
- File from **10 days ago** → Compress to `.gz`
- File from **35 days ago** → Delete completely

---

## 🔍 Line-by-Line Explanation

### Line 1: Shebang
```bash
#!/bin/bash
```

**What it does**: Tells the system "run this with bash"

**Simple analogy**: Like saying "Dear Computer, please use bash to read this recipe"

**Why it matters**: Without this, the system might not know how to run your script

---

### Line 2-3: Variables (Setup)
```bash
LOG_DIR="/var/log/myapp"
LOG_FILE="/var/log/myapp/log_rotation.log"
```

**What it does**: Creates shortcuts for paths you'll use a lot

**Think of it like**:
- `LOG_DIR` = "The folder where all my app logs live"
- `LOG_FILE` = "My diary where I write what I did"

**Why variables?** Instead of typing `/var/log/myapp` 10 times, just type `$LOG_DIR`

**Example**:
```bash
# Without variables (messy):
find /var/log/myapp -type f -name "*.log"
echo "Done" >> /var/log/myapp/log_rotation.log

# With variables (clean):
find $LOG_DIR -type f -name "*.log"
echo "Done" >> $LOG_FILE
```

---

### Line 4-8: Safety Check
```bash
if [ ! -d "$LOG_DIR" ]; then
    echo "[$(date)] ERROR: Log directory $LOG_DIR does not exist!" >> "$LOG_FILE"
    exit 1
fi
```

**What it does**: Checks if the log folder exists before doing anything

**Breaking it down**:
```bash
if [ ! -d "$LOG_DIR" ]; then
```
- `if` → Start a condition
- `[ ]` → Test something
- `!` → NOT (reverse the result)
- `-d` → Check if it's a Directory
- `"$LOG_DIR"` → The path to check

**Translation**: "If the directory does NOT exist, then..."

```bash
echo "[$(date)] ERROR: Log directory does not exist!" >> "$LOG_FILE"
```
- `echo` → Print a message
- `$(date)` → Current date/time (e.g., "Sun Feb 22 10:30:45")
- `>>` → Append to file (don't overwrite)

```bash
exit 1
```
- Stop the script immediately
- `1` means "failed" (0 means success)

**Why this matters**: Prevents script from breaking if folder doesn't exist

---

### Line 9-10: Compress Old Logs (7-30 days)
```bash
find "$LOG_DIR" -type f -name "*.log" -mtime +7 -mtime -30 ! -name "*.gz" -exec gzip {} \; -exec echo "[$(date)] Compressed: {}" >> "$LOG_FILE" \;
```

**This is the most complex line!** Let's break it into pieces:

#### Part 1: The `find` Command Setup
```bash
find "$LOG_DIR"
```
**What**: Look in the `/var/log/myapp` folder

#### Part 2: File Type Filter
```bash
-type f
```
**What**: Only find FILES (not folders)
**Why**: We don't want to compress directories

#### Part 3: Name Pattern
```bash
-name "*.log"
```
**What**: Only files ending in `.log`
**Examples**:
- ✅ `app.log` → matches
- ✅ `error.log` → matches
- ❌ `app.txt` → doesn't match

#### Part 4: Age Filter (Older than 7 days)
```bash
-mtime +7
```
**What**: Files modified MORE than 7 days ago
**The `+` means "greater than"**

**Visual**:
```
Today ←─ 7 days ─→ Older files
 0       7         8, 9, 10...
         │         
         └─ +7 means "after this line"
```

#### Part 5: Age Filter (Newer than 30 days)
```bash
-mtime -30
```
**What**: Files modified LESS than 30 days ago
**The `-` means "less than"**

**Combined with previous**:
```
+7 AND -30 = between 7 and 30 days old
```

**Visual**:
```
Today ←─ 7 ──┼── 30 ─→ Really old
       ✅     │     ❌
    [7-30 days range]
```

#### Part 6: Exclude Already Compressed
```bash
! -name "*.gz"
```
**What**: Files NOT ending in `.gz`
**Why**: Don't compress files that are already compressed!

**The `!` means "NOT"**

#### Part 7: Execute Compression
```bash
-exec gzip {} \;
```
**What**: For each file found, run `gzip` on it

**Breaking it down**:
- `-exec` → Execute a command
- `gzip` → The compression program
- `{}` → Placeholder for the filename
- `\;` → End of command (the backslash escapes the semicolon)

**Example**:
```bash
# If find discovers: /var/log/myapp/app.log
# It runs: gzip /var/log/myapp/app.log
# Result: /var/log/myapp/app.log.gz
```

#### Part 8: Log the Action
```bash
-exec echo "[$(date)] Compressed: {}" >> "$LOG_FILE" \;
```
**What**: Write a line to the log file saying what was compressed

**Example output in log_rotation.log**:
```
[Sun Feb 22 10:30:45 2024] Compressed: /var/log/myapp/app.log
[Sun Feb 22 10:30:45 2024] Compressed: /var/log/myapp/error.log
```

---

### Line 11-12: Delete Very Old Compressed Logs (30+ days)
```bash
find "$LOG_DIR" -type f -name "*.gz" -mtime +30 -exec rm -f {} \; -exec echo "[$(date)] Deleted: {}" >> "$LOG_FILE" \;
```

**Similar to previous, but simpler!**

**Breaking it down**:
```bash
find "$LOG_DIR"          # Look in log directory
-type f                  # Files only
-name "*.gz"             # Only compressed files
-mtime +30               # Older than 30 days
-exec rm -f {} \;        # Delete the file
-exec echo "..." \;      # Log the deletion
```

**What `rm -f` means**:
- `rm` → Remove (delete)
- `-f` → Force (don't ask for confirmation)

**Visual**:
```
Today ←───── 30 days ─────→ These get deleted
                           *.gz files
```

---

### Line 13-14: Delete Very Old Uncompressed Logs (Safety Net)
```bash
find "$LOG_DIR" -type f -name "*.log" -mtime +30 -exec rm -f {} \; -exec echo "[$(date)] Deleted (uncompressed): {}" >> "$LOG_FILE" \;
```

**What**: Deletes `.log` files older than 30 days that somehow weren't compressed

**Why this line?**: Safety net in case compression failed

**Scenarios it catches**:
- Script was interrupted during compression
- File was created while script was running
- Permissions issue prevented compression

---

### Line 15-16: Success Message
```bash
echo "[$(date)] Log rotation completed successfully." >> "$LOG_FILE"
```

**What**: Writes a "finished" message to the log

**Example output**:
```
[Sun Feb 22 10:30:45 2024] Compressed: /var/log/myapp/app.log
[Sun Feb 22 10:30:46 2024] Deleted: /var/log/myapp/old.log.gz
[Sun Feb 22 10:30:46 2024] Log rotation completed successfully.
```

---

## 🎓 Complete Example Walkthrough

### Before Script Runs
```
/var/log/myapp/
├── app.log              (today)          → Keep as-is
├── error.log            (3 days old)     → Keep as-is
├── access.log           (10 days old)    → Compress to .gz
├── debug.log            (15 days old)    → Compress to .gz
├── old.log.gz           (35 days old)    → Delete
└── ancient.log          (40 days old)    → Delete
```

### After Script Runs
```
/var/log/myapp/
├── app.log              (unchanged)
├── error.log            (unchanged)
├── access.log.gz        (newly compressed)
├── debug.log.gz         (newly compressed)
└── log_rotation.log     (script log)
```

**Deleted**: `old.log.gz`, `ancient.log`

---

## 🔧 Improved Version with Comments

```bash
#!/bin/bash
#===============================================================================
# Script: log_cleanup.sh
# Purpose: Compress old logs and delete very old logs to save disk space
# Author: DevOps Team
# Date: 2024-02-22
#===============================================================================

#───────────────────────────────────────────────────────────────────────────────
# Configuration
#───────────────────────────────────────────────────────────────────────────────
LOG_DIR="/var/log/myapp"              # Where app logs are stored
LOG_FILE="/var/log/log_rotation.log"  # Where we log our actions
COMPRESS_DAYS=7                        # Compress logs older than this
DELETE_DAYS=30                         # Delete logs older than this

#───────────────────────────────────────────────────────────────────────────────
# Safety Check: Verify log directory exists
#───────────────────────────────────────────────────────────────────────────────
if [ ! -d "$LOG_DIR" ]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: Directory $LOG_DIR not found!" >> "$LOG_FILE"
    exit 1
fi

#───────────────────────────────────────────────────────────────────────────────
# Step 1: Compress logs between 7-30 days old
#───────────────────────────────────────────────────────────────────────────────
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting log compression..." >> "$LOG_FILE"

find "$LOG_DIR" \
    -type f \
    -name "*.log" \
    -mtime +$COMPRESS_DAYS \
    -mtime -$DELETE_DAYS \
    ! -name "*.gz" \
    -exec gzip {} \; \
    -exec echo "[$(date '+%Y-%m-%d %H:%M:%S')] Compressed: {}" >> "$LOG_FILE" \;

#───────────────────────────────────────────────────────────────────────────────
# Step 2: Delete compressed logs older than 30 days
#───────────────────────────────────────────────────────────────────────────────
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Deleting old compressed logs..." >> "$LOG_FILE"

find "$LOG_DIR" \
    -type f \
    -name "*.gz" \
    -mtime +$DELETE_DAYS \
    -exec rm -f {} \; \
    -exec echo "[$(date '+%Y-%m-%d %H:%M:%S')] Deleted: {}" >> "$LOG_FILE" \;

#───────────────────────────────────────────────────────────────────────────────
# Step 3: Delete uncompressed logs older than 30 days (safety net)
#───────────────────────────────────────────────────────────────────────────────
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cleaning up old uncompressed logs..." >> "$LOG_FILE"

find "$LOG_DIR" \
    -type f \
    -name "*.log" \
    -mtime +$DELETE_DAYS \
    -exec rm -f {} \; \
    -exec echo "[$(date '+%Y-%m-%d %H:%M:%S')] Deleted (uncompressed): {}" >> "$LOG_FILE" \;

#───────────────────────────────────────────────────────────────────────────────
# Completion
#───────────────────────────────────────────────────────────────────────────────
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Log rotation completed successfully." >> "$LOG_FILE"
echo "─────────────────────────────────────────────────────────────────────" >> "$LOG_FILE"

# Show summary
COMPRESSED=$(grep "Compressed:" "$LOG_FILE" | tail -10 | wc -l)
DELETED=$(grep "Deleted:" "$LOG_FILE" | tail -10 | wc -l)
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Summary: $COMPRESSED compressed, $DELETED deleted" >> "$LOG_FILE"
```

---

## ⏰ Setting Up Daily Cron Job

### Step 1: Make Script Executable
```bash
chmod +x /usr/local/bin/log_cleanup.sh
```

**What `chmod +x` does**: Makes the file runnable as a program

### Step 2: Test the Script Manually
```bash
# Dry run (see what would happen without doing it)
find /var/log/myapp -type f -name "*.log" -mtime +7 -mtime -30 ! -name "*.gz"

# Actually run the script
sudo /usr/local/bin/log_cleanup.sh

# Check the log
cat /var/log/log_rotation.log
```

### Step 3: Add to Crontab
```bash
# Edit root's crontab
sudo crontab -e

# Add this line (runs daily at 2 AM):
0 2 * * * /usr/local/bin/log_cleanup.sh
```

**Cron format explained**:
```
0 2 * * *
│ │ │ │ │
│ │ │ │ └─ Day of week (0-7, Sunday = 0 or 7)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)
```

**Common schedules**:
```bash
0 2 * * *      # Daily at 2 AM
0 */6 * * *    # Every 6 hours
0 0 * * 0      # Weekly (Sunday midnight)
0 3 1 * *      # Monthly (1st day at 3 AM)
*/15 * * * *   # Every 15 minutes
```

### Step 4: Verify Cron Job
```bash
# List current cron jobs
sudo crontab -l

# Check if cron is running
systemctl status cron

# View cron logs
grep CRON /var/log/syslog | tail -20
```

---

## 🧪 Testing the Script

### Create Test Files
```bash
# Create test directory
mkdir -p /tmp/test-logs

# Create files with different ages
touch -d "2 days ago" /tmp/test-logs/recent.log
touch -d "10 days ago" /tmp/test-logs/old.log
touch -d "35 days ago" /tmp/test-logs/ancient.log
touch -d "35 days ago" /tmp/test-logs/ancient.log.gz

# List files with ages
ls -lht /tmp/test-logs/
```

### Run Script on Test Data
```bash
# Modify script to use test directory
LOG_DIR="/tmp/test-logs"

# Run script
./log_cleanup.sh

# Check results
ls -lh /tmp/test-logs/
cat /var/log/log_rotation.log
```

---

## ⚠️ Common Issues & Solutions

### Issue 1: Permission Denied
```bash
# Error:
# ./log_cleanup.sh: Permission denied

# Solution:
chmod +x log_cleanup.sh
```

### Issue 2: Script Doesn't Run from Cron
```bash
# Cron has minimal PATH
# Fix: Use absolute paths

# Wrong:
gzip file.log

# Right:
/usr/bin/gzip file.log
```

### Issue 3: Date Command Not Found
```bash
# Some systems don't have date in PATH
# Fix: Use absolute path
/bin/date '+%Y-%m-%d %H:%M:%S'
```

### Issue 4: Log File Grows Too Large
```bash
# Add log rotation for the script's own log
# In script, add:
if [ -f "$LOG_FILE" ] && [ $(stat -f%z "$LOG_FILE" 2>/dev/null || stat -c%s "$LOG_FILE") -gt 10485760 ]; then
    mv "$LOG_FILE" "$LOG_FILE.old"
    gzip "$LOG_FILE.old"
fi
```

---

## 📊 Monitoring & Alerts

### Add Email Notification
```bash
# At end of script:
SUMMARY="Compressed: $COMPRESSED files, Deleted: $DELETED files"
echo "$SUMMARY" | mail -s "Log Rotation Report" admin@example.com
```

### Add Slack Notification
```bash
# At end of script:
WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
MESSAGE="Log rotation completed: $COMPRESSED compressed, $DELETED deleted"

curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$MESSAGE\"}" \
    "$WEBHOOK"
```

---

## 🎯 Key Concepts Summary

| Concept | Simple Explanation | Example |
|---------|-------------------|---------|
| `find` | Search for files | `find . -name "*.log"` |
| `-mtime +7` | Older than 7 days | Files modified 8+ days ago |
| `-mtime -30` | Newer than 30 days | Files modified 0-29 days ago |
| `gzip` | Compress file | `app.log` → `app.log.gz` |
| `{}` | Placeholder for filename | Each found file |
| `\;` | End of command | Marks where `-exec` ends |
| `>>` | Append to file | Don't overwrite |
| `exit 1` | Exit with error | Something went wrong |

---

## 🎓 Interview Answer Template

**When explaining this script in an interview**:

**1. Purpose** (10 seconds):
> "This script manages log files by compressing old ones to save space and deleting very old ones to prevent disk full issues."

**2. Key Logic** (20 seconds):
> "It uses `find` with `-mtime` to select files by age: compress files 7-30 days old with `gzip`, and delete files over 30 days old with `rm`. All actions are logged for auditing."

**3. Safety Measures** (10 seconds):
> "It checks if the directory exists before running, logs all actions, and has a safety net to catch uncompressed old files."

**4. Automation** (10 seconds):
> "Scheduled daily at 2 AM via cron. I'd test it manually first, verify the cron syntax, and monitor the log file to ensure it's working."

**5. Improvements** (bonus points):
> "I'd add email alerts on failures, disk space checks before/after, and make the retention days configurable via environment variables."

---

**🎯 Bottom Line**: The script finds files by age, compresses the old ones, deletes the very old ones, and logs everything. Simple but powerful!
