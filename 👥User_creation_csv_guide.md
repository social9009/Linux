# 👥 Automated User Creation from CSV - Complete Guide

## 🎯 The Basic Script (Your Answer)

```bash
#!/bin/bash
INPUT="users.csv"

if [[ ! -f "$INPUT" ]]; then
  echo "CSV file not found!"
  exit 1
fi

tail -n +2 "$INPUT" | while IFS=',' read -r username password; do
  if id "$username" &>/dev/null; then
    echo "User '$username' already exists. Skipping..."
    continue
  fi
  
  useradd "$username"
  echo "${username}:${password}" | chpasswd
  chage -d 0 "$username"
  
  echo "User '$username' created successfully."
done
```

**This is a great start!** Let me show you how to make it production-ready.

---

## 🔧 Enhanced Production-Ready Script

```bash
#!/bin/bash
#===============================================================================
# Script: create_users.sh
# Description: Create Linux users from CSV file with secure defaults
# Author: DevOps Team
# Usage: sudo ./create_users.sh [-f file.csv] [-g group] [-s shell] [-v]
#===============================================================================

set -euo pipefail  # Exit on error, undefined vars, pipe failures
IFS=$'\n\t'        # Better word splitting

#───────────────────────────────────────────────────────────────────────────────
# Configuration
#───────────────────────────────────────────────────────────────────────────────
CSV_FILE="users.csv"
DEFAULT_SHELL="/bin/bash"
DEFAULT_GROUP=""
VERBOSE=false
DRY_RUN=false
LOG_FILE="/var/log/user_creation_$(date +%Y%m%d_%H%M%S).log"
HOME_DIR_BASE="/home"

#───────────────────────────────────────────────────────────────────────────────
# Colors for output
#───────────────────────────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

#───────────────────────────────────────────────────────────────────────────────
# Functions
#───────────────────────────────────────────────────────────────────────────────

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

error() {
  echo -e "${RED}[ERROR]${NC} $*" | tee -a "$LOG_FILE" >&2
}

success() {
  echo -e "${GREEN}[SUCCESS]${NC} $*" | tee -a "$LOG_FILE"
}

warning() {
  echo -e "${YELLOW}[WARNING]${NC} $*" | tee -a "$LOG_FILE"
}

info() {
  if [[ "$VERBOSE" == true ]]; then
    echo -e "${BLUE}[INFO]${NC} $*" | tee -a "$LOG_FILE"
  fi
}

usage() {
  cat << EOF
Usage: $0 [OPTIONS]

Create Linux users from CSV file.

OPTIONS:
  -f FILE     CSV file path (default: users.csv)
  -g GROUP    Add users to this group
  -s SHELL    Default shell (default: /bin/bash)
  -d DIR      Home directory base (default: /home)
  -n          Dry run (show what would be done)
  -v          Verbose output
  -h          Show this help

CSV Format:
  username,password[,full_name[,groups]]

Example:
  alice,Password@123,Alice Smith,wheel,developers
  bob,Secure@456,Bob Jones

EOF
  exit 0
}

validate_username() {
  local username="$1"
  
  # Check length (1-32 chars)
  if [[ ${#username} -lt 1 || ${#username} -gt 32 ]]; then
    return 1
  fi
  
  # Check valid characters (alphanumeric, dash, underscore)
  if [[ ! "$username" =~ ^[a-z_][a-z0-9_-]*$ ]]; then
    return 1
  fi
  
  return 0
}

validate_password() {
  local password="$1"
  
  # Minimum length check
  if [[ ${#password} -lt 8 ]]; then
    warning "Password too short (min 8 chars)"
    return 1
  fi
  
  # Check complexity (optional - adjust as needed)
  if [[ ! "$password" =~ [A-Z] ]] || \
     [[ ! "$password" =~ [a-z] ]] || \
     [[ ! "$password" =~ [0-9] ]]; then
    warning "Password doesn't meet complexity requirements"
    return 1
  fi
  
  return 0
}

user_exists() {
  id "$1" &>/dev/null
}

group_exists() {
  getent group "$1" &>/dev/null
}

create_user() {
  local username="$1"
  local password="$2"
  local full_name="${3:-}"
  local additional_groups="${4:-}"
  
  info "Processing user: $username"
  
  # Validate username
  if ! validate_username "$username"; then
    error "Invalid username format: $username"
    return 1
  fi
  
  # Check if user exists
  if user_exists "$username"; then
    warning "User '$username' already exists. Skipping..."
    return 0
  fi
  
  # Validate password
  if ! validate_password "$password"; then
    error "Invalid password for user: $username"
    return 1
  fi
  
  # Dry run mode
  if [[ "$DRY_RUN" == true ]]; then
    info "[DRY RUN] Would create user: $username"
    info "[DRY RUN] - Shell: $DEFAULT_SHELL"
    info "[DRY RUN] - Groups: $DEFAULT_GROUP $additional_groups"
    return 0
  fi
  
  # Build useradd command
  local useradd_cmd="useradd"
  useradd_cmd+=" -m"  # Create home directory
  useradd_cmd+=" -s $DEFAULT_SHELL"
  useradd_cmd+=" -d $HOME_DIR_BASE/$username"
  
  # Add full name if provided
  if [[ -n "$full_name" ]]; then
    useradd_cmd+=" -c \"$full_name\""
  fi
  
  # Add to groups
  local all_groups=""
  if [[ -n "$DEFAULT_GROUP" ]]; then
    all_groups="$DEFAULT_GROUP"
  fi
  if [[ -n "$additional_groups" ]]; then
    if [[ -n "$all_groups" ]]; then
      all_groups+=","
    fi
    all_groups+="$additional_groups"
  fi
  
  if [[ -n "$all_groups" ]]; then
    # Verify groups exist
    IFS=',' read -ra GROUPS <<< "$all_groups"
    for group in "${GROUPS[@]}"; do
      if ! group_exists "$group"; then
        warning "Group '$group' doesn't exist. Creating..."
        groupadd "$group" || error "Failed to create group: $group"
      fi
    done
    useradd_cmd+=" -G $all_groups"
  fi
  
  # Create user
  info "Creating user with command: $useradd_cmd $username"
  eval "$useradd_cmd $username"
  
  if [[ $? -ne 0 ]]; then
    error "Failed to create user: $username"
    return 1
  fi
  
  # Set password
  echo "${username}:${password}" | chpasswd
  
  if [[ $? -ne 0 ]]; then
    error "Failed to set password for: $username"
    # Delete the user since password failed
    userdel -r "$username" 2>/dev/null
    return 1
  fi
  
  # Force password change on first login
  chage -d 0 "$username"
  
  # Set password aging policies
  chage -M 90 "$username"  # Max 90 days
  chage -m 1 "$username"   # Min 1 day between changes
  chage -W 7 "$username"   # Warn 7 days before expiry
  
  # Set secure home directory permissions
  chmod 700 "$HOME_DIR_BASE/$username"
  
  success "User '$username' created successfully"
  return 0
}

#───────────────────────────────────────────────────────────────────────────────
# Parse Arguments
#───────────────────────────────────────────────────────────────────────────────
while getopts "f:g:s:d:nvh" opt; do
  case $opt in
    f) CSV_FILE="$OPTARG" ;;
    g) DEFAULT_GROUP="$OPTARG" ;;
    s) DEFAULT_SHELL="$OPTARG" ;;
    d) HOME_DIR_BASE="$OPTARG" ;;
    n) DRY_RUN=true ;;
    v) VERBOSE=true ;;
    h) usage ;;
    *) usage ;;
  esac
done

#───────────────────────────────────────────────────────────────────────────────
# Pre-flight Checks
#───────────────────────────────────────────────────────────────────────────────

# Check if running as root
if [[ $EUID -ne 0 ]]; then
  error "This script must be run as root (use sudo)"
  exit 1
fi

# Check if CSV file exists
if [[ ! -f "$CSV_FILE" ]]; then
  error "CSV file not found: $CSV_FILE"
  exit 1
fi

# Validate CSV format (check header)
HEADER=$(head -n 1 "$CSV_FILE")
if [[ ! "$HEADER" =~ ^username,password ]]; then
  error "Invalid CSV format. Expected header: username,password[,full_name[,groups]]"
  exit 1
fi

# Check if required commands exist
REQUIRED_CMDS=("useradd" "chpasswd" "chage" "groupadd")
for cmd in "${REQUIRED_CMDS[@]}"; do
  if ! command -v "$cmd" &>/dev/null; then
    error "Required command not found: $cmd"
    exit 1
  fi
done

#───────────────────────────────────────────────────────────────────────────────
# Main Execution
#───────────────────────────────────────────────────────────────────────────────

log "════════════════════════════════════════════════════════════"
log "Starting user creation from: $CSV_FILE"
log "Log file: $LOG_FILE"
log "Dry run: $DRY_RUN"
log "════════════════════════════════════════════════════════════"

# Statistics
TOTAL=0
SUCCESS=0
SKIPPED=0
FAILED=0

# Process CSV file
while IFS=',' read -r username password full_name groups; do
  # Skip header
  if [[ "$username" == "username" ]]; then
    continue
  fi
  
  # Skip empty lines
  if [[ -z "$username" ]]; then
    continue
  fi
  
  # Trim whitespace
  username=$(echo "$username" | xargs)
  password=$(echo "$password" | xargs)
  full_name=$(echo "$full_name" | xargs)
  groups=$(echo "$groups" | xargs)
  
  ((TOTAL++))
  
  if create_user "$username" "$password" "$full_name" "$groups"; then
    ((SUCCESS++))
  else
    if user_exists "$username"; then
      ((SKIPPED++))
    else
      ((FAILED++))
    fi
  fi
  
done < <(tail -n +2 "$CSV_FILE")

#───────────────────────────────────────────────────────────────────────────────
# Summary
#───────────────────────────────────────────────────────────────────────────────

log "════════════════════════════════════════════════════════════"
log "User Creation Summary:"
log "  Total processed: $TOTAL"
log "  Successfully created: $SUCCESS"
log "  Skipped (already exist): $SKIPPED"
log "  Failed: $FAILED"
log "════════════════════════════════════════════════════════════"

# Exit with error if any failed
if [[ $FAILED -gt 0 ]]; then
  exit 1
fi

exit 0
```

---

## 📊 Enhanced CSV Format Support

```csv
username,password,full_name,groups
alice,Password@123,Alice Smith,developers,wheel
bob,Secure@456,Bob Jones,developers
carol,DevOps@789,Carol White,operations,docker
dave,Admin@321,Dave Brown,
```

**Features**:
- Username (required)
- Password (required)
- Full name (optional) - stored in GECOS field
- Groups (optional) - comma-separated list

---

## 🔒 Security Enhancements

### 1. Password Validation
```bash
validate_password() {
  local password="$1"
  
  # Length check
  if [[ ${#password} -lt 8 ]]; then
    return 1
  fi
  
  # Complexity: uppercase, lowercase, digit, special char
  if [[ ! "$password" =~ [A-Z] ]] || \
     [[ ! "$password" =~ [a-z] ]] || \
     [[ ! "$password" =~ [0-9] ]] || \
     [[ ! "$password" =~ [^a-zA-Z0-9] ]]; then
    return 1
  fi
  
  return 0
}
```

### 2. Username Validation
```bash
validate_username() {
  # Must start with letter or underscore
  # Can contain: letters, digits, dash, underscore
  # Length: 1-32 characters
  if [[ ! "$1" =~ ^[a-z_][a-z0-9_-]{0,31}$ ]]; then
    return 1
  fi
  return 0
}
```

### 3. Password Aging Policies
```bash
chage -M 90 "$username"  # Maximum 90 days before change required
chage -m 1 "$username"   # Minimum 1 day between changes
chage -W 7 "$username"   # Warning 7 days before expiry
chage -I 30 "$username"  # Account inactive 30 days after password expires
```

### 4. Secure Home Directory
```bash
chmod 700 "/home/$username"  # Only user can access
```

---

## 🛠️ Usage Examples

### Basic Usage
```bash
# Using default CSV file (users.csv)
sudo ./create_users.sh

# Specify custom CSV file
sudo ./create_users.sh -f employees.csv

# Verbose output
sudo ./create_users.sh -v
```

### Advanced Usage
```bash
# Add all users to 'developers' group
sudo ./create_users.sh -g developers

# Use custom shell
sudo ./create_users.sh -s /bin/zsh

# Dry run (test without creating)
sudo ./create_users.sh -n -v

# Custom home directory base
sudo ./create_users.sh -d /data/homes

# Combine options
sudo ./create_users.sh -f staff.csv -g employees -s /bin/bash -v
```

---

## 📝 Creating the CSV File

### Method 1: Manual Creation
```bash
cat > users.csv << 'EOF'
username,password,full_name,groups
alice,Password@123,Alice Smith,developers,wheel
bob,Secure@456,Bob Jones,developers
carol,DevOps@789,Carol White,operations,docker
EOF
```

### Method 2: From Excel/Google Sheets
1. Create spreadsheet with columns: username, password, full_name, groups
2. Export as CSV
3. Upload to server

### Method 3: Generate Programmatically
```bash
#!/bin/bash
# generate_users_csv.sh

cat > users.csv << 'EOF'
username,password,full_name,groups
EOF

# Generate users
for i in {1..10}; do
  username="user$i"
  password="Temp@$(openssl rand -base64 12 | tr -d '/+=')"
  echo "$username,$password,User $i,developers" >> users.csv
done
```

---

## 🧪 Testing the Script

### Create Test CSV
```bash
cat > test_users.csv << 'EOF'
username,password,full_name,groups
testuser1,Test@123,Test User One,testgroup
testuser2,Test@456,Test User Two,testgroup
EOF
```

### Dry Run First
```bash
sudo ./create_users.sh -f test_users.csv -n -v
```

### Execute
```bash
sudo ./create_users.sh -f test_users.csv -v
```

### Verify Users Created
```bash
# Check user exists
id testuser1

# Check groups
groups testuser1

# Check home directory
ls -la /home/testuser1

# Check password aging
chage -l testuser1
```

### Test Login
```bash
# Switch to user
su - testuser1
# Will be prompted to change password

# Or via SSH
ssh testuser1@localhost
```

### Cleanup
```bash
# Delete test users
sudo userdel -r testuser1
sudo userdel -r testuser2
sudo groupdel testgroup 2>/dev/null
```

---

## 🔐 Best Practices

### 1. Secure Password Storage
```bash
# DON'T: Keep passwords in plain text CSV
# DO: Generate random passwords and email them

#!/bin/bash
while read -r username _; do
  if [[ "$username" == "username" ]]; then continue; fi
  
  # Generate secure random password
  password=$(openssl rand -base64 16)
  
  # Email to user (requires mailutils)
  echo "Your temporary password: $password" | \
    mail -s "New Account" "${username}@company.com"
    
  echo "${username}:${password}" | chpasswd
done < users_without_passwords.csv
```

### 2. Audit Logging
```bash
# Log all actions
log "Creating user: $username"
log "Added to groups: $groups"

# Send to syslog
logger -t user_creation "Created user: $username"
```

### 3. Backup Before Changes
```bash
# Backup /etc/passwd and /etc/shadow
cp /etc/passwd /etc/passwd.backup.$(date +%Y%m%d)
cp /etc/shadow /etc/shadow.backup.$(date +%Y%m%d)
```

### 4. Idempotency
```bash
# Script can be run multiple times safely
if user_exists "$username"; then
  warning "User exists. Skipping..."
  continue
fi
```

---

## 📋 Common Issues & Solutions

### Issue 1: Permission Denied
```bash
# Error: Only root can add users
# Solution: Run with sudo
sudo ./create_users.sh
```

### Issue 2: User Already Exists
```bash
# Script skips existing users automatically
# To recreate: delete first
sudo userdel -r username
```

### Issue 3: Invalid Characters in CSV
```bash
# Problem: Windows line endings (CRLF)
# Solution: Convert to Unix format
dos2unix users.csv

# Or using sed
sed -i 's/\r$//' users.csv
```

### Issue 4: Password Rejected
```bash
# Check password requirements
grep -E '^(PASS_MIN_LEN|PASS_MAX_DAYS)' /etc/login.defs

# Adjust script validation accordingly
```

---

## 🚀 Production Deployment

### Step 1: Install Script
```bash
# Copy to system location
sudo cp create_users.sh /usr/local/bin/
sudo chmod 755 /usr/local/bin/create_users.sh

# Create log directory
sudo mkdir -p /var/log/user_management
```

### Step 2: Security
```bash
# Secure the CSV file (contains passwords!)
chmod 600 users.csv
chown root:root users.csv

# Store in secure location
sudo mv users.csv /root/user_management/
```

### Step 3: Run
```bash
# Execute
sudo /usr/local/bin/create_users.sh -f /root/user_management/users.csv -v

# Check logs
sudo tail -f /var/log/user_creation_*.log
```

### Step 4: Cleanup
```bash
# Securely delete CSV with passwords
sudo shred -vfz -n 10 /root/user_management/users.csv
```

---

## 🎓 Interview Answer Template

**Perfect answer structure**:

**Basic** (20s):
> "I'd create a bash script that reads the CSV file using `read -r` with `IFS=','` to parse each line. For each user, I'd use `useradd` to create the account, `chpasswd` to set the password, and `chage -d 0` to force password change on first login."

**Show Best Practices** (20s):
> "I'd add validation for usernames and passwords, check if users already exist to make the script idempotent, add error handling with `set -euo pipefail`, and log all actions for audit purposes. I'd also ensure the script runs with proper permissions using a root check."

**Security** (20s):
> "For security, I'd validate password complexity, set password aging policies with `chage -M 90 -m 1 -W 7`, secure home directories with `chmod 700`, and immediately shred the CSV file after use since it contains plaintext passwords. In production, I'd generate random passwords and email them instead."

**Production Ready** (bonus):
> "I'd add features like dry-run mode, verbose logging, support for additional fields like full name and groups, ability to add users to multiple groups, and comprehensive error handling. I'd also create unit tests to verify the script works correctly before running in production."

---

## 📊 Quick Reference

```bash
# Basic user creation
useradd username

# With home directory
useradd -m username

# With shell
useradd -m -s /bin/bash username

# With groups
useradd -m -G developers,wheel username

# Set password
echo "username:password" | chpasswd

# Force password change
chage -d 0 username

# Password aging
chage -M 90 -m 1 -W 7 username

# Check user info
id username
chage -l username
groups username
```

---

**🎯 Bottom Line**: Reading CSV and creating users is straightforward, but production scripts need validation, error handling, logging, security checks, and idempotency. Always test with dry-run first and securely handle password files!
