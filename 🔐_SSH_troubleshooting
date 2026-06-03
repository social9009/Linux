# 🔐 SSH Connection Troubleshooting - Complete Guide

## 🎯 Understanding SSH Errors

**Different error messages mean different things**:

```bash
# Error 1: Connection timeout
ssh: connect to host 1.2.3.4 port 22: Connection timed out
→ Firewall blocking or instance unreachable

# Error 2: Connection refused
ssh: connect to host 1.2.3.4 port 22: Connection refused
→ SSH service not running or not listening

# Error 3: Permission denied (publickey)
Permission denied (publickey)
→ Key authentication failed

# Error 4: Host key verification failed
Host key verification failed
→ Server fingerprint changed

# Error 5: No route to host
ssh: connect to host 1.2.3.4 port 22: No route to host
→ Network routing issue
```

---

## 🔍 Systematic Troubleshooting (10 Steps)

### Step 1: Reproduce with Verbose Output (30 seconds)

```bash
# Enable maximum verbosity (-vvv)
ssh -vvv -i my-key.pem ec2-user@1.2.3.4

# This shows:
# - DNS resolution
# - TCP connection attempt
# - SSH handshake
# - Key exchange
# - Authentication attempt
# - Exact failure point
```

#### Interpreting Verbose Output

**Example 1: Network Issue**
```
debug1: Connecting to 1.2.3.4 [1.2.3.4] port 22.
debug1: connect to address 1.2.3.4 port 22: Connection timed out
→ Network/firewall problem
```

**Example 2: SSH Service Down**
```
debug1: Connecting to 1.2.3.4 [1.2.3.4] port 22.
debug1: connect to address 1.2.3.4 port 22: Connection refused
→ SSH daemon not running
```

**Example 3: Wrong Key**
```
debug1: Offering public key: /home/user/.ssh/my-key.pem RSA SHA256:ABC123...
debug1: Authentications that can continue: publickey
debug1: No more authentication methods to try.
Permission denied (publickey)
→ Key mismatch or wrong user
```

---

### Step 2: Check Instance Status (1 minute)

#### AWS Console Method
```
EC2 → Instances → Select instance → Check:
- Instance State: running ✅
- Status Checks: 2/2 passed ✅
- System reachability: passed ✅
- Instance reachability: passed ✅
```

#### AWS CLI Method
```bash
# Check instance status
aws ec2 describe-instance-status --instance-ids i-xxxxxxxxx

# Output:
{
    "InstanceStatuses": [{
        "InstanceState": {"Name": "running"},
        "SystemStatus": {"Status": "ok"},
        "InstanceStatus": {"Status": "ok"}
    }]
}

# Get instance details
aws ec2 describe-instances --instance-ids i-xxxxxxxxx \
    --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress,PrivateIpAddress]' \
    --output table
```

#### Common Status Issues

**Issue 1: Instance Stopped**
```bash
State: stopped
→ Start the instance
aws ec2 start-instances --instance-ids i-xxxxxxxxx
```

**Issue 2: Status Checks Failed**
```bash
SystemStatus: impaired
→ Usually AWS infrastructure issue, may need instance stop/start

InstanceStatus: impaired
→ OS-level issue, check logs via console or restart
```

**Issue 3: Instance Terminating**
```bash
State: shutting-down / terminated
→ Cannot recover, launch new instance
```

---

### Step 3: Test Network Connectivity (1 minute)

#### Ping Test (ICMP)
```bash
# Test if host is reachable
ping -c 4 1.2.3.4

# If times out:
# - Security group may block ICMP
# - Instance may be down
# - Network routing issue
```

#### TCP Port Test
```bash
# Method 1: telnet
telnet 1.2.3.4 22
# Output if working: "Connected to 1.2.3.4"
# Output if blocked: "Connection refused" or timeout

# Method 2: nc (netcat)
nc -zv 1.2.3.4 22
# Output: Connection to 1.2.3.4 22 port [tcp/ssh] succeeded!

# Method 3: nmap
nmap -p 22 1.2.3.4
# Output: 22/tcp open  ssh

# Method 4: curl (checks if port is listening)
curl -v telnet://1.2.3.4:22
```

---

### Step 4: Verify Security Group Rules (2 minutes)

#### Check via AWS Console
```
EC2 → Instances → Select instance → Security tab → Security groups
→ Check Inbound rules
```

**Required rule**:
```
Type: SSH
Protocol: TCP
Port: 22
Source: Your IP (203.0.113.0/32) or 0.0.0.0/0
```

#### Check via AWS CLI
```bash
# Get security group ID
aws ec2 describe-instances --instance-ids i-xxxxxxxxx \
    --query 'Reservations[0].Instances[0].SecurityGroups[*].GroupId' \
    --output text

# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# Look for:
{
    "IpPermissions": [{
        "FromPort": 22,
        "ToPort": 22,
        "IpProtocol": "tcp",
        "IpRanges": [{"CidrIp": "0.0.0.0/0"}]  # or your IP
    }]
}
```

#### Common Security Group Issues

**Issue 1: SSH Not Allowed**
```bash
# Add SSH rule
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxxx \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0  # Or your specific IP
```

**Issue 2: Wrong IP Range**
```bash
# Current rule: 203.0.113.0/32 (specific IP)
# Your actual IP: 198.51.100.45
→ Either add your IP or use 0.0.0.0/0 (less secure)
```

**Issue 3: Rule Accidentally Removed**
```bash
# Check security group history
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-xxxxxxxxx \
    --max-results 10
```

---

### Step 5: Verify Network ACLs (1 minute)

Network ACLs are stateless (must allow both inbound and outbound).

#### Check via Console
```
VPC → Network ACLs → Select ACL → Inbound/Outbound rules
```

**Required rules**:
```
Inbound:
Rule #    Type        Protocol    Port Range    Source        Allow/Deny
100       SSH (22)    TCP (6)     22            0.0.0.0/0     ALLOW

Outbound:
Rule #    Type        Protocol    Port Range    Destination   Allow/Deny
100       Custom TCP  TCP (6)     1024-65535    0.0.0.0/0     ALLOW
```

#### Check via CLI
```bash
# Find subnet's NACL
aws ec2 describe-network-acls \
    --filters "Name=association.subnet-id,Values=subnet-xxxxxxxx"

# Check for SSH allow rule
aws ec2 describe-network-acls --network-acl-ids acl-xxxxxxxx \
    --query 'NetworkAcls[0].Entries[?RuleNumber==`100`]'
```

---

### Step 6: Verify Public IP Address (1 minute)

#### Check Current IP
```bash
# Get public IP
aws ec2 describe-instances --instance-ids i-xxxxxxxxx \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text

# Get Elastic IP (if attached)
aws ec2 describe-addresses \
    --filters "Name=instance-id,Values=i-xxxxxxxxx"
```

#### Common IP Issues

**Issue 1: No Public IP**
```bash
# Check if instance has public IP
PublicIpAddress: null
→ Instance in private subnet or auto-assign disabled

# Solutions:
# A) Allocate and associate Elastic IP
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id i-xxxxxxxxx --allocation-id eipalloc-xxxxxxxx

# B) Use bastion host or VPN
```

**Issue 2: IP Changed After Restart**
```bash
# Public IPs change when instance stops/starts
# Old IP: 1.2.3.4
# New IP: 5.6.7.8

# Solution: Use Elastic IP (doesn't change)
```

**Issue 3: DNS Cached**
```bash
# Clear DNS cache
# Linux:
sudo systemd-resolve --flush-caches

# macOS:
sudo dscacheutil -flushcache

# Or use IP directly instead of hostname
ssh -i key.pem ec2-user@1.2.3.4
```

---

### Step 7: Validate SSH Key & Permissions (2 minutes)

#### Check Key File Permissions
```bash
# Must be 400 or 600
ls -la ~/.ssh/my-key.pem
# -r-------- (400) ✅
# -rw------- (600) ✅
# -rw-r--r-- (644) ❌

# Fix permissions
chmod 400 ~/.ssh/my-key.pem
```

#### Verify Correct Key
```bash
# Get key fingerprint from file
ssh-keygen -lf ~/.ssh/my-key.pem
# Output: 2048 SHA256:ABC123... my-key.pem (RSA)

# Get key fingerprint from AWS
aws ec2 describe-key-pairs --key-name my-key
# Compare fingerprints - they should match
```

#### Verify Correct Username
```bash
# Common usernames by AMI:
# Amazon Linux 2:  ec2-user
# Ubuntu:          ubuntu
# Debian:          admin
# RHEL:            ec2-user
# CentOS:          centos
# Fedora:          fedora
# SUSE:            ec2-user

# Try different users
ssh -i key.pem ec2-user@1.2.3.4
ssh -i key.pem ubuntu@1.2.3.4
ssh -i key.pem admin@1.2.3.4
```

#### Test with Different Key Formats
```bash
# If using PuTTY on Windows, convert .ppk to .pem
puttygen my-key.ppk -O private-openssh -o my-key.pem

# If using newer OpenSSH format
ssh-keygen -p -f my-key.pem -m pem -P "" -N ""
```

---

### Step 8: Use EC2 Instance Connect (AWS Only)

#### Via AWS Console
```
EC2 → Instances → Select instance → Connect → EC2 Instance Connect
→ Opens browser-based terminal
```

**Requirements**:
- Amazon Linux 2 or Ubuntu 16.04+
- Instance has public IP
- Security group allows SSH from AWS IP ranges
- IAM permissions for EC2 Instance Connect

#### Via AWS CLI
```bash
# Generate temporary SSH key and connect
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-xxxxxxxxx \
    --instance-os-user ec2-user \
    --ssh-public-key file://~/.ssh/temp_key.pub

ssh -i ~/.ssh/temp_key ec2-user@1.2.3.4
```

#### Once Connected, Check SSH Service
```bash
# Check SSH daemon status
sudo systemctl status sshd

# View SSH logs
sudo tail -100 /var/log/auth.log     # Debian/Ubuntu
sudo tail -100 /var/log/secure       # RHEL/CentOS

# Check sshd config
sudo sshd -T | grep -E 'port|permitroot|pubkey|password'

# Restart SSH service
sudo systemctl restart sshd
```

---

### Step 9: Check SSH Service Configuration (Advanced)

If you have console access (EC2 Instance Connect, Serial Console, or another server):

#### Verify SSH Daemon Running
```bash
# Check if sshd is running
sudo systemctl status sshd

# If not running
sudo systemctl start sshd
sudo systemctl enable sshd

# Check if listening on port 22
sudo ss -tulnp | grep :22
# Should show: LISTEN  0  128  0.0.0.0:22
```

#### Check SSHD Configuration
```bash
# Test config syntax
sudo sshd -t

# View effective configuration
sudo sshd -T

# Common config issues
sudo grep -E '^(Port|PermitRootLogin|PubkeyAuthentication|PasswordAuthentication)' /etc/ssh/sshd_config
```

**Required settings**:
```
Port 22                          # Or custom port
PubkeyAuthentication yes
PermitRootLogin no              # For ec2-user/ubuntu
PasswordAuthentication no       # Use keys, not passwords
```

#### Check Authorized Keys
```bash
# View authorized keys
cat ~/.ssh/authorized_keys

# Check permissions (must be 600 or 700)
ls -la ~/.ssh/
# drwx------ .ssh         (700) ✅
# -rw------- authorized_keys (600) ✅

# Fix permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R ec2-user:ec2-user ~/.ssh  # Or correct user
```

#### View SSH Connection Attempts
```bash
# Real-time log watching
sudo tail -f /var/log/auth.log     # Debian/Ubuntu
sudo tail -f /var/log/secure       # RHEL/CentOS

# Filter for failures
sudo grep "Failed" /var/log/auth.log
sudo grep "refused" /var/log/auth.log

# Check for specific IP
sudo grep "203.0.113.45" /var/log/auth.log
```

---

### Step 10: Use Systems Manager Session Manager (No SSH Needed)

SSM Session Manager works even without SSH access.

#### Prerequisites
```bash
# 1. Instance has IAM role with AmazonSSMManagedInstanceCore policy
# 2. SSM agent is running (pre-installed on most AMIs)
# 3. Instance can reach SSM endpoints (public IP or VPC endpoints)
```

#### Connect via Console
```
Systems Manager → Session Manager → Start session → Select instance
```

#### Connect via CLI
```bash
# Install Session Manager plugin
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

# Start session
aws ssm start-session --target i-xxxxxxxxx

# Once connected, check SSH
sudo systemctl status sshd
sudo tail -50 /var/log/auth.log
```

---

## 🛠️ Recovery Methods (When All Else Fails)

### Method 1: EC2 Serial Console

**Enable via Console**:
```
EC2 → Account Settings → EC2 Serial Console → Manage → Allow
```

**Requirements**:
- Root account access
- Password-based authentication set on instance
- Only available in select regions

**Connect**:
```
EC2 → Instances → Select instance → Connect → EC2 serial console
```

### Method 2: User Data Script (Auto-Fix on Next Boot)

```bash
# Stop instance
aws ec2 stop-instances --instance-ids i-xxxxxxxxx

# Get current user data
aws ec2 describe-instance-attribute \
    --instance-id i-xxxxxxxxx \
    --attribute userData

# Add fix script as user data
cat > fix-ssh.sh <<'EOF'
#!/bin/bash
# Add new SSH key
echo "ssh-rsa AAAAB3NzaC1... new-key" >> /home/ec2-user/.ssh/authorized_keys
chmod 600 /home/ec2-user/.ssh/authorized_keys
chown ec2-user:ec2-user /home/ec2-user/.ssh/authorized_keys

# Fix SSH config
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
systemctl restart sshd
EOF

# Base64 encode
USER_DATA=$(base64 -w 0 fix-ssh.sh)

# Update user data
aws ec2 modify-instance-attribute \
    --instance-id i-xxxxxxxxx \
    --user-data "$USER_DATA"

# Start instance
aws ec2 start-instances --instance-ids i-xxxxxxxxx

# Clean up user data after fix
aws ec2 modify-instance-attribute \
    --instance-id i-xxxxxxxxx \
    --user-data ""
```

### Method 3: EBS Volume Swap (Nuclear Option)

See detailed steps in my PEM recovery guide, but summary:

```bash
# 1. Stop instance
aws ec2 stop-instances --instance-ids i-xxxxxxxxx

# 2. Detach root volume
aws ec2 detach-volume --volume-id vol-xxxxxxxxx

# 3. Attach to rescue instance
aws ec2 attach-volume --volume-id vol-xxxxxxxxx \
    --instance-id i-rescue-instance \
    --device /dev/sdf

# 4. Mount and fix
sudo mount /dev/xvdf1 /mnt
sudo nano /mnt/home/ec2-user/.ssh/authorized_keys
# Add your new public key
sudo umount /mnt

# 5. Detach and reattach to original instance
aws ec2 detach-volume --volume-id vol-xxxxxxxxx
aws ec2 attach-volume --volume-id vol-xxxxxxxxx \
    --instance-id i-xxxxxxxxx \
    --device /dev/xvda

# 6. Start original instance
aws ec2 start-instances --instance-ids i-xxxxxxxxx
```

---

## 🎯 Real-World Scenarios

### Scenario 1: "Permission denied (publickey)"

**Symptoms**: Connection established but authentication fails

**Diagnosis**:
```bash
ssh -vvv -i key.pem ec2-user@1.2.3.4
# Output:
# debug1: Offering public key: key.pem RSA SHA256:ABC123...
# debug1: Authentications that can continue: publickey
# debug1: No more authentication methods to try.
```

**Common Causes**:
1. Wrong key file
2. Wrong username
3. Key permissions wrong (not 400/600)
4. Authorized_keys file corrupted

**Solution**:
```bash
# 1. Verify key fingerprint matches
ssh-keygen -lf key.pem
aws ec2 describe-key-pairs --key-name my-key

# 2. Try different usernames
ssh -i key.pem ubuntu@1.2.3.4

# 3. Fix permissions
chmod 400 key.pem

# 4. Use EC2 Instance Connect to check authorized_keys
cat ~/.ssh/authorized_keys
```

---

### Scenario 2: "Connection timed out"

**Symptoms**: Connection hangs then times out

**Diagnosis**:
```bash
ssh -vvv -i key.pem ec2-user@1.2.3.4
# Output:
# debug1: Connecting to 1.2.3.4 [1.2.3.4] port 22.
# (hangs for 60+ seconds)
# ssh: connect to host 1.2.3.4 port 22: Connection timed out
```

**Common Causes**:
1. Security group blocks port 22
2. Network ACL blocks traffic
3. Wrong IP address
4. Instance in private subnet

**Solution**:
```bash
# 1. Test with telnet
telnet 1.2.3.4 22
# If times out → firewall issue

# 2. Check security group
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# 3. Verify IP is correct
aws ec2 describe-instances --instance-ids i-xxxxxxxxx \
    --query 'Reservations[0].Instances[0].PublicIpAddress'

# 4. Add SSH rule to security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxxx \
    --protocol tcp --port 22 --cidr 0.0.0.0/0
```

---

### Scenario 3: "Connection refused"

**Symptoms**: Connection reaches instance but SSH daemon isn't responding

**Diagnosis**:
```bash
ssh -vvv -i key.pem ec2-user@1.2.3.4
# Output:
# debug1: Connecting to 1.2.3.4 [1.2.3.4] port 22.
# debug1: connect to address 1.2.3.4 port 22: Connection refused
```

**Common Causes**:
1. SSH daemon not running
2. SSH listening on different port
3. Instance OS crashed

**Solution**:
```bash
# Use EC2 Instance Connect or SSM to check
sudo systemctl status sshd
sudo systemctl start sshd

# Check what's listening
sudo ss -tulnp | grep :22

# Check SSH config
sudo grep -E '^Port' /etc/ssh/sshd_config
```

---

## 📋 Troubleshooting Checklist

```
□ Test with verbose SSH: ssh -vvv
□ Check instance status: running + 2/2 checks passed
□ Test network: ping and telnet/nc to port 22
□ Verify security group: SSH (22) allowed from your IP
□ Check NACL: Inbound 22 and outbound ephemeral ports
□ Confirm public IP: not null and correct IP
□ Validate key file: permissions 400/600
□ Try correct username: ec2-user, ubuntu, admin
□ Test with EC2 Instance Connect
□ Check SSM connectivity
□ Review SSH daemon logs
□ Verify authorized_keys file
□ Check sshd_config settings
□ Use recovery methods if needed
```

---

## 🎓 Interview Answer Template

**Perfect 60-second response**:

**1. Reproduce** (5s):
> "First, I'd SSH with verbose mode `ssh -vvv -i key.pem ec2-user@IP` to see exactly where it fails—network connection, SSH handshake, or authentication."

**2. Layer 1: Network** (15s):
> "I'd check if the instance is running and reachable with `aws ec2 describe-instance-status` and test connectivity with `telnet IP 22` or `nc -zv IP 22`. Then verify security group allows SSH from my IP and NACLs aren't blocking port 22."

**3. Layer 2: Instance** (15s):
> "If network is fine, I'd use EC2 Instance Connect or SSM Session Manager to access the instance and check if SSH daemon is running with `systemctl status sshd`, review logs in `/var/log/auth.log`, and verify the config with `sshd -t`."

**4. Layer 3: Authentication** (10s):
> "For authentication issues, I'd verify key fingerprint matches with `ssh-keygen -lf key.pem`, check key permissions are 400, try different usernames (ec2-user, ubuntu), and inspect `~/.ssh/authorized_keys` file."

**5. Recovery** (10s):
> "If all else fails, I'd use user data script to auto-fix on boot, or detach the root volume, mount it on a rescue instance, add a new key to authorized_keys, and reattach."

**6. Real Example** (5s):
> "I once had SSH fail after a security group was accidentally updated. Used EC2 Instance Connect to verify SSH was running, then added the SSH rule back to the security group and it worked immediately."

---

**🎯 Bottom Line**: SSH issues are usually network (security groups), service (sshd not running), or keys (wrong key/user). Check in that order, use EC2 Instance Connect/SSM as backup access.
