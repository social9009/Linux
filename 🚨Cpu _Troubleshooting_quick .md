# 🔥 Linux High CPU Utilization - Complete Troubleshooting Guide

## 🚨 The Problem: Server is Slow Due to High CPU

High CPU usage can be caused by:
- **Application issues**: Infinite loops, memory leaks, inefficient code
- **System issues**: Zombie processes, kernel bugs, hardware problems  
- **External factors**: DDoS attacks, crypto miners, scheduled jobs
- **Resource contention**: Too many processes for available cores

---

## 🎯 6-Phase Troubleshooting Process

### Phase 1: Initial Assessment (30 seconds)

```bash
# Check load average
uptime
# Output: 14:23:45 up 5 days, 3:21, 2 users, load average: 8.42, 6.73, 4.21

# Get CPU count
nproc
# Output: 4

# Rule: Load > cores = overloaded (8.42 on 4 cores = 200% overload)

# Quick CPU breakdown
mpstat 1 3
# Look for: %idle < 20% = stressed
```

### Phase 2: Identify Top Processes

```bash
# Method 1: top (classic)
top -o %CPU

# Method 2: htop (better UI)
htop  # Press P to sort by CPU

# Method 3: ps (scriptable)
ps aux --sort=-%cpu | head -20

# Method 4: pidstat (time-series)
pidstat -u 1 5
```

### Phase 3: Deep Analysis

See detailed scenarios below...

### Phase 4: Root Cause Investigation

See scenario-specific sections...

### Phase 5: Immediate Remediation

See solution sections...

### Phase 6: Permanent Fix

See optimization sections...

---

## 🔍 Real-World Scenarios

### Scenario 1: Application Process at 90% CPU

**Quick diagnosis**:
```bash
# Identify process
ps -p 1234 -o pid,cmd,%cpu,etime

# What is it doing?
sudo strace -p 1234 -c    # System call summary
sudo perf top -p 1234     # Function-level CPU
```

**Solutions by language**:

**Python**:
```bash
sudo py-spy top --pid 1234
sudo py-spy record -o profile.svg --pid 1234 --duration 30
```

**Java**:
```bash
jstat -gcutil 1234 1000    # Check GC
jstack 1234                # Thread dump
```

**Node.js**:
```bash
kill -SIGUSR1 1234         # Generate profile
clinic doctor -- node app.js
```

### Scenario 2: CPU Spike Every Hour (Cron Job)

**Diagnosis**:
```bash
grep CRON /var/log/syslog | tail -50
ps aux | grep cron
systemctl list-timers
```

**Fix**:
```bash
# Add timeout to cron
timeout 300 /path/to/script.sh

# Prevent overlap with flock
flock -n /tmp/script.lock -c '/path/to/script.sh'
```

### Scenario 3: Crypto Miner

**Detection**:
```bash
# Suspicious process names
ps aux | grep -E 'minerd|xmrig|cpuminer|ethminer'

# Network connections to mining pools
sudo netstat -tunap | grep -E 'pool|stratum'
```

**Removal**:
```bash
pkill -9 minerd
find / -name "xmrig" -delete 2>/dev/null
```

---

## 🛠️ Immediate Fixes

### Kill Process
```bash
kill -9 1234                     # Force kill
pkill -f "process-name"          # Kill by name
systemctl restart nginx          # Restart service
```

### Limit CPU
```bash
# Nice (lower priority)
nice -n 19 ./task.sh
renice -n 19 -p 1234

# cpulimit (hard limit)
cpulimit -p 1234 -l 50           # 50% of one core

# systemd (permanent)
[Service]
CPUQuota=50%
```

---

## 📊 Profiling Tools

```bash
# System-wide profiling
perf record -a -g sleep 30
perf report

# Application profiling  
strace -c -p 1234                # System calls
lsof -p 1234                     # Open files
pmap 1234                        # Memory map
```

---

## 🎯 Quick Reference

| Command | Purpose |
|---------|---------|
| `uptime` | Load average |
| `top -o %CPU` | Top processes |
| `ps aux --sort=-%cpu` | CPU hogs |
| `pidstat -u 1 5` | Time-series |
| `perf top` | Live profiling |
| `kill -9 PID` | Force kill |
| `nice -n 19` | Lower priority |
| `cpulimit -l 50` | Hard limit |

---

## 🎓 Interview Answer Template

**1. Assess**: "Check `uptime` for load average, then `top` to identify processes"

**2. Diagnose**: "Use `ps aux`, `pidstat`, and `strace` to understand what the process is doing"

**3. Root cause**: "Usually infinite loops, GC thrashing, bad queries, or overlapping cron jobs"

**4. Fix**: "Kill/restart process, use `nice`/`cpulimit` to limit, or fix code"

**5. Prevent**: "Optimize code, add caching, set up CPU monitoring with alerts"

**6. Example**: "I once debugged a Python worker hitting 100% CPU using `py-spy`, found an infinite retry loop, and fixed it by adding a max retry count"

**Bottom line**: Identify → Analyze → Fix → Prevent
