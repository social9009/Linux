# 🔐 EC2 PEM File Recovery - Complete Guide

## ❌ Can You Restore a Lost PEM File?

**NO** - PEM files (private keys) are **NEVER stored on AWS** after creation. Once lost, they cannot be recovered.

AWS generates the key pair and gives you the private key **only once** during creation. After that, AWS only stores the public key fingerprint.

---

## ✅ How to Regain Access Without PEM File

There are **4 methods** to recover access, ranked from easiest to most complex:

```
Method 1: EC2 Instance Connect (Easiest - if supported)
Method 2: Systems Manager (SSM) Session Manager
Method 3: User Data Script (stops/starts instance)
Method 4: EBS Volume Swap (Manual rescue - most reliable)
```

---

## 🚀 Method 1: EC2 Instance Connect (Fastest)

### Prerequisites
- Amazon Linux 2 or Ubuntu 16.04+
- Instance has public IP
- Security group allows SSH (port 22)
- IAM permissions for EC2 Instance Connect

### Steps

#### 1. Use AWS Console
```
EC2 Console → Instances → Select instance → Connect → EC2 Instance Connect
```
This opens a browser-based terminal **without needing PEM file**.

#### 2. Add New SSH Key
Once connected via browser:
```bash
# Generate new key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/new-key

# Add public key to authorized_keys
cat ~/.ssh/new-key.pub >> ~/.ssh/authorized_keys

# Download the private key
cat ~/.ssh/new-key
```

Copy the private key content to your local machine as `new-key.pem`:
```bash
# On your local machine
chmod 400 new-key.pem
ssh -i new-key.pem ec2-user@<instance-ip>
```

### ✅ Pros
- Fastest method
- No instance restart required
- Works immediately

### ❌ Cons
- Only works on Amazon Linux 2, Ubuntu 16.04+, and a few other AMIs
- Requires instance to have public IP
- Requires security group allowing SSH

---

## 🔧 Method 2: Systems Manager (SSM) Session Manager

### Prerequisites
- SSM agent installed (pre-installed on most recent AMIs)
- Instance has IAM role with `AmazonSSMManagedInstanceCore` policy
- No public IP required

### Steps

#### 1. Attach IAM Role (if not already attached)
```bash
# Create IAM role with SSM policy
aws iam create-role --role-name EC2-SSM-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach SSM policy
aws iam attach-role-policy --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Attach role to instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxx \
  --iam-instance-profile Name=EC2-SSM-Role
```

**trust-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### 2. Connect via Session Manager
```bash
# Using AWS CLI
aws ssm start-session --target i-xxxxxxxxx

# Or use AWS Console
Systems Manager → Session Manager → Start session → Select instance
```

#### 3. Add New SSH Key
```bash
# Switch to ec2-user
sudo su - ec2-user

# Add new public key
echo "ssh-rsa AAAAB3NzaC1yc2E... your-new-key" >> ~/.ssh/authorized_keys

# Now you can SSH with new key
ssh -i new-key.pem ec2-user@<instance-ip>
```

### ✅ Pros
- Works without SSH access
- No public IP required
- No security group changes needed
- Works on private subnets

### ❌ Cons
- Requires IAM role attachment (may need instance stop/start)
- SSM agent must be installed and running

---

## 🔄 Method 3: User Data Script (Automated Key Injection)

### Prerequisites
- Ability to stop and start the instance (data loss if instance store)
- Works on any Linux AMI

### Steps

#### 1. Stop the Instance
```bash
aws ec2 stop-instances --instance-ids i-xxxxxxxxx
```

#### 2. Create New Key Pair
```bash
# Create new key pair
aws ec2 create-key-pair --key-name recovery-key \
  --query 'KeyMaterial' --output text > recovery-key.pem

chmod 400 recovery-key.pem

# Get the public key
ssh-keygen -y -f recovery-key.pem > recovery-key.pub
```

#### 3. Modify User Data to Inject Key
```bash
# Get the public key content
PUB_KEY=$(cat recovery-key.pub)

# Create user data script
cat > user-data.sh <<EOF
#!/bin/bash
echo "$PUB_KEY" >> /home/ec2-user/.ssh/authorized_keys
EOF

# Convert to base64
USER_DATA=$(base64 -w 0 user-data.sh)

# Update instance user data
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxx \
  --user-data "$USER_DATA"
```

#### 4. Start Instance
```bash
aws ec2 start-instances --instance-ids i-xxxxxxxxx
```

#### 5. SSH with New Key
```bash
ssh -i recovery-key.pem ec2-user@<instance-ip>
```

#### 6. Clean Up User Data (Important!)
```bash
# Remove user data to prevent key duplication on next boot
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxx \
  --user-data ""
```

### ✅ Pros
- Fully automated
- Works on any Linux AMI
- No manual volume manipulation

### ❌ Cons
- Requires instance restart
- Lost instance store data (if any)
- User data runs on every boot unless cleaned up

---

## 🛠️ Method 4: EBS Volume Swap (Manual Rescue)

### Prerequisites
- Ability to stop instance
- Understanding of volume management
- Works on ALL instances

This is the **most reliable method** but requires manual steps.

### Detailed Step-by-Step Process

#### Step 1: Create New Key Pair
```bash
# Create new key pair locally (more secure than AWS-generated)
ssh-keygen -t rsa -b 4096 -f recovery-key -N ""

# This creates:
# - recovery-key (private key)
# - recovery-key.pub (public key)

chmod 400 recovery-key
```

#### Step 2: Stop the Affected Instance
```bash
# Stop the instance
aws ec2 stop-instances --instance-ids i-XXXXXXXXX

# Wait until stopped
aws ec2 wait instance-stopped --instance-ids i-XXXXXXXXX
```

#### Step 3: Identify and Detach Root Volume
```bash
# Get root volume ID
ROOT_VOLUME=$(aws ec2 describe-instances \
  --instance-ids i-XXXXXXXXX \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId' \
  --output text)

echo "Root volume: $ROOT_VOLUME"

# Detach volume
aws ec2 detach-volume --volume-id $ROOT_VOLUME

# Wait for detachment
aws ec2 wait volume-available --volume-ids $ROOT_VOLUME
```

#### Step 4: Create or Use Temporary Instance
```bash
# Option A: Use existing instance in same AZ
# Option B: Create temporary rescue instance

# Create rescue instance (if needed)
RESCUE_INSTANCE=$(aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --instance-type t3.micro \
  --key-name your-working-key \
  --subnet-id subnet-xxxxxxxx \
  --query 'Instances[0].InstanceId' \
  --output text)

# Wait for rescue instance to be running
aws ec2 wait instance-running --instance-ids $RESCUE_INSTANCE
```

#### Step 5: Attach Volume to Rescue Instance
```bash
# Attach volume as secondary device
aws ec2 attach-volume \
  --volume-id $ROOT_VOLUME \
  --instance-id $RESCUE_INSTANCE \
  --device /dev/sdf

# Wait for attachment
aws ec2 wait volume-in-use --volume-ids $ROOT_VOLUME
```

#### Step 6: SSH into Rescue Instance and Mount Volume
```bash
# SSH to rescue instance
ssh -i your-working-key.pem ec2-user@<rescue-instance-ip>

# Find the device name (may be /dev/xvdf or /dev/nvme1n1)
lsblk

# Output example:
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda    202:0    0   8G  0 disk
# └─xvda1 202:1    0   8G  0 part /
# xvdf    202:80   0  20G  0 disk
# └─xvdf1 202:81   0  20G  0 part

# Create mount point
sudo mkdir /mnt/recovery

# Mount the partition (usually partition 1)
sudo mount /dev/xvdf1 /mnt/recovery

# Verify mount
df -h | grep recovery
```

#### Step 7: Inject New SSH Key
```bash
# Check if authorized_keys exists
sudo ls -la /mnt/recovery/home/ec2-user/.ssh/

# Method A: Edit authorized_keys directly
sudo nano /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Add your new public key:
# ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... recovery-key

# Method B: Append programmatically
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... recovery-key" | \
  sudo tee -a /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Verify key was added
sudo cat /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Fix permissions (important!)
sudo chmod 700 /mnt/recovery/home/ec2-user/.ssh
sudo chmod 600 /mnt/recovery/home/ec2-user/.ssh/authorized_keys
```

#### Step 8: Unmount and Detach Volume
```bash
# Unmount the volume
sudo umount /mnt/recovery

# Exit from rescue instance
exit

# Detach volume from rescue instance
aws ec2 detach-volume --volume-id $ROOT_VOLUME

# Wait for detachment
aws ec2 wait volume-available --volume-ids $ROOT_VOLUME
```

#### Step 9: Re-attach Volume to Original Instance
```bash
# Re-attach to original instance as root volume
aws ec2 attach-volume \
  --volume-id $ROOT_VOLUME \
  --instance-id i-XXXXXXXXX \
  --device /dev/xvda

# Wait for attachment
aws ec2 wait volume-in-use --volume-ids $ROOT_VOLUME
```

#### Step 10: Start Original Instance
```bash
# Start the instance
aws ec2 start-instances --instance-ids i-XXXXXXXXX

# Wait until running
aws ec2 wait instance-running --instance-ids i-XXXXXXXXX

# Get public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids i-XXXXXXXXX \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Instance IP: $PUBLIC_IP"
```

#### Step 11: SSH with New Key
```bash
# Test SSH connection
ssh -i recovery-key ec2-user@$PUBLIC_IP

# Success!
```

#### Step 12: Clean Up Rescue Instance (if created)
```bash
# Terminate rescue instance
aws ec2 terminate-instances --instance-ids $RESCUE_INSTANCE
```

### ✅ Pros
- Works on **any** EC2 instance
- Most reliable method
- Complete control over the process

### ❌ Cons
- Most complex
- Requires instance downtime
- Manual volume manipulation

---

## 📊 Method Comparison Table

| Method | Complexity | Downtime | Works On | Prerequisites |
|--------|-----------|----------|----------|--------------|
| **EC2 Instance Connect** | ⭐ Easy | ❌ None | Amazon Linux 2, Ubuntu 16.04+ | Public IP |
| **SSM Session Manager** | ⭐⭐ Medium | ⚠️ Minimal* | All AMIs with SSM | IAM role |
| **User Data Script** | ⭐⭐ Medium | ✅ Yes | All Linux AMIs | Stop/Start access |
| **EBS Volume Swap** | ⭐⭐⭐ Hard | ✅ Yes | **All instances** | Volume access |

*May require restart to attach IAM role

---

## 🔒 Security Best Practices

### 1. Prevent PEM Loss in the Future

#### Store Keys Securely
```bash
# Encrypt PEM file with GPG
gpg --symmetric --cipher-algo AES256 my-key.pem
# Creates my-key.pem.gpg

# Decrypt when needed
gpg --decrypt my-key.pem.gpg > my-key.pem
chmod 400 my-key.pem
```

#### Use AWS Secrets Manager
```bash
# Store PEM file in Secrets Manager
aws secretsmanager create-secret \
  --name ec2-pem-keys/production-key \
  --secret-string file://my-key.pem

# Retrieve later
aws secretsmanager get-secret-value \
  --secret-id ec2-pem-keys/production-key \
  --query SecretString \
  --output text > recovered-key.pem

chmod 400 recovered-key.pem
```

### 2. Create Multiple Access Methods

#### Add Backup User with Different Key
```bash
# On the EC2 instance
sudo adduser backup-admin
sudo mkdir /home/backup-admin/.ssh
sudo chmod 700 /home/backup-admin/.ssh

# Add backup public key
echo "ssh-rsa AAAAB3... backup-key" | \
  sudo tee /home/backup-admin/.ssh/authorized_keys

sudo chmod 600 /home/backup-admin/.ssh/authorized_keys
sudo chown -R backup-admin:backup-admin /home/backup-admin/.ssh

# Grant sudo access
echo "backup-admin ALL=(ALL) NOPASSWD:ALL" | \
  sudo tee /etc/sudoers.d/backup-admin
```

### 3. Enable SSM by Default

#### Create Launch Template with SSM Role
```bash
# Create IAM role for all instances
aws iam create-role --role-name Default-EC2-SSM-Role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name Default-EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name Default-EC2-SSM-Profile

aws iam add-role-to-instance-profile \
  --instance-profile-name Default-EC2-SSM-Profile \
  --role-name Default-EC2-SSM-Role

# Use in launch template
aws ec2 create-launch-template \
  --launch-template-name secure-instance-template \
  --version-description "v1" \
  --launch-template-data '{
    "IamInstanceProfile": {
      "Name": "Default-EC2-SSM-Profile"
    }
  }'
```

### 4. Use EC2 Instance Connect Endpoint (New!)

AWS now offers **EC2 Instance Connect Endpoint** which allows private instance access without bastion hosts:

```bash
# Create EIC Endpoint
aws ec2 create-instance-connect-endpoint \
  --subnet-id subnet-xxxxxxxx \
  --security-group-ids sg-xxxxxxxx

# Connect to private instance
aws ec2-instance-connect ssh \
  --instance-id i-xxxxxxxxx
```

---

## 🚨 Common Mistakes to Avoid

### ❌ Mistake 1: Wrong Device Name
```bash
# Wrong - Device may be renamed by kernel
sudo mount /dev/sdf1 /mnt/recovery

# Correct - Use lsblk to verify actual device name
lsblk
sudo mount /dev/xvdf1 /mnt/recovery  # or /dev/nvme1n1p1
```

### ❌ Mistake 2: Wrong User Directory
```bash
# Wrong - Ubuntu uses 'ubuntu' user
sudo nano /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Correct - Check user directory first
ls -la /mnt/recovery/home/
sudo nano /mnt/recovery/home/ubuntu/.ssh/authorized_keys
```

### ❌ Mistake 3: Incorrect Permissions
```bash
# After adding key, fix permissions
sudo chmod 700 /mnt/recovery/home/ec2-user/.ssh
sudo chmod 600 /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Verify ownership
sudo ls -la /mnt/recovery/home/ec2-user/.ssh/
```

### ❌ Mistake 4: Forgetting to Detach Before Re-attach
```bash
# Must wait for volume to be available
aws ec2 wait volume-available --volume-ids vol-xxxxxxxx

# Then attach
aws ec2 attach-volume --volume-id vol-xxxxxxxx --instance-id i-xxxxxxxx --device /dev/xvda
```

---

## 🧪 Testing Your Recovery Process

### Simulate PEM Loss (Safe Test)
```bash
# 1. Create test instance
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --instance-type t3.micro \
  --key-name test-key \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=test-recovery}]'

# 2. "Lose" the PEM file (rename it)
mv test-key.pem test-key.pem.backup

# 3. Practice recovery using one of the methods above

# 4. Clean up
aws ec2 terminate-instances --instance-ids i-xxxxxxxx
mv test-key.pem.backup test-key.pem
```

---

## 📋 Recovery Checklist

```
□ Verify instance is in the correct region
□ Stop instance (if using volume swap method)
□ Identify root volume ID
□ Note the original device name (/dev/xvda)
□ Create rescue instance in SAME availability zone
□ Attach volume to rescue instance
□ Mount volume and verify user directory
□ Add new public key to authorized_keys
□ Fix permissions (700 for .ssh, 600 for authorized_keys)
□ Unmount and detach volume
□ Re-attach to original instance as root device
□ Start original instance
□ Test SSH with new key
□ Clean up rescue instance
```

---

## 🎓 Interview Tips

When answering this question in an interview, demonstrate:

1. **Understanding of fundamentals**:
   - "PEM files contain the private key and cannot be recovered from AWS"
   - "AWS only stores the public key fingerprint"

2. **Multiple recovery approaches**:
   - "I'd first try EC2 Instance Connect or SSM if available"
   - "For production, I'd use the EBS volume swap method for reliability"

3. **Prevention mindset**:
   - "I always enable SSM on instances"
   - "We store PEM files encrypted in Secrets Manager"
   - "Each instance has a backup key pair"

4. **Real experience**:
   - "I've recovered access 3 times using the volume swap method"
   - "We have a runbook for this exact scenario"

---

**🎯 Bottom Line**: You cannot restore a lost PEM file, but you have multiple recovery options. The EBS volume swap method works 100% of the time on any instance.
