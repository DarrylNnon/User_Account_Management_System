# User Account Management System
Setting up a User Account Management System on Linux involves automating user creation, enforcing password policies, managing account expirations, and documenting account management policies.I provide a comprehensive guide to implement each part step-by-step.

## Project Overview: User Account Management System

In this project, we'll:
1. **Automate user creation with specific permissions**.
2. **Enforce a password policy**.
3. **Automatically manage expired accounts**.
4. **Document account management policies**.

### Prerequisites
1. **Linux Server Access**: Root or sudo privileges are required.
2. **Basic Scripting Knowledge**: Familiarity with Bash scripting.
3. **Knowledge of User Management Commands**: Commands like `useradd`, `usermod`, `chage`, and `passwd`.


### Step 1: Script for Automated User Creation with Specific Permissions

We’ll create a script to automate user creation, specify user group membership, set up home directories, and assign permissions.

#### Step 1.1: Create the User Creation Script

**Script: `create_user.sh`**

```bash
#!/bin/bash

# Check if script is run as root
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root" 
   exit 1
fi

# Define variables
USERNAME=$1
USER_GROUP="developers"
DEFAULT_SHELL="/bin/bash"
HOMEDIR="/home/$USERNAME"

# Check if username was provided
if [[ -z "$USERNAME" ]]; then
  echo "Usage: ./create_user.sh <username>"
  exit 1
fi

# Create a new user with the specified shell and home directory
useradd -m -d "$HOMEDIR" -s "$DEFAULT_SHELL" -G "$USER_GROUP" "$USERNAME"

# Check if user was successfully created
if [[ $? -eq 0 ]]; then
  echo "User $USERNAME created successfully with home directory $HOMEDIR and added to group $USER_GROUP."
else
  echo "Failed to create user $USERNAME."
  exit 1
fi

# Set default permissions on the user's home directory
chmod 750 "$HOMEDIR"
echo "Permissions for $USERNAME set to 750."

# Prompt for an initial password
echo "Please set the initial password for $USERNAME:"
passwd "$USERNAME"

echo "User creation and setup complete for $USERNAME."
```

- **Explanation**:
  - This script creates a new user with a specified group (`developers`), home directory, shell (`/bin/bash`), and home directory permissions.
  - The `chmod 750` command sets permissions, so only the user and group can access the directory.
  - The user’s initial password must be set upon creation.

- **Execution**:
  ```bash
  chmod +x create_user.sh
  sudo ./create_user.sh newuser
  ```

### Step 2: Integrate Password Policy Enforcement

To enforce password policies, configure `/etc/login.defs` for basic policies and use `pam_pwquality` for advanced policies.

#### Step 2.1: Configure Basic Password Policies

Edit `/etc/login.defs` to set basic parameters such as password expiration, minimum password length, and password complexity.

```bash
# Example configurations in /etc/login.defs

PASS_MAX_DAYS   90        # Password expires after 90 days
PASS_MIN_DAYS   7         # Minimum 7 days before changing password
PASS_WARN_AGE   14        # 14 days warning before password expires
```

#### Step 2.2: Configure Advanced Password Policies with PAM

Edit `/etc/security/pwquality.conf` to enforce password complexity (if `pam_pwquality` is installed).

**Example configuration in `/etc/security/pwquality.conf`:**

```plaintext
minlen = 8                  # Minimum password length
minclass = 3                # Require at least 3 character classes (e.g., uppercase, lowercase, digits)
retry = 3                   # User gets 3 attempts to enter a compliant password
```

#### Step 2.3: Enforce Policy in PAM Configuration

Edit `/etc/pam.d/common-password` (or `/etc/pam.d/password-auth` on Red Hat-based systems) to include `pam_pwquality`:

```plaintext
password requisite pam_pwquality.so retry=3
password required pam_unix.so use_authtok md5 shadow
```

- **Explanation**:
  - The password policies enforce complexity, expiration, and warning periods, ensuring secure user accounts.


### Step 3: Manage Expired Accounts Automatically

We’ll set up an automated system to disable accounts when they reach expiration and delete them after a specified period.

#### Step 3.1: Set Expiration Dates for Users

You can set expiration dates for new users or update existing users with the `chage` command.

**Example: Set expiration for a user**

```bash
# Set expiration date to 90 days from now
sudo chage -E $(date -d "+90 days" +%F) newuser
```

#### Step 3.2: Script for Automatic Expiration Management

**Script: `manage_expired_accounts.sh`**

This script disables accounts that have expired and sends a report.

```bash
#!/bin/bash

# Report file
REPORT="/var/log/expired_accounts.log"

# Find expired accounts and lock them
echo "Checking for expired accounts..."
for user in $(cut -f1 -d: /etc/shadow); do
  if [[ $(chage -l $user | grep "Account expires" | awk -F": " '{print $2}') == "expired" ]]; then
    usermod -L $user
    echo "Account $user has expired and is locked." >> $REPORT
  fi
done

echo "Expired account management completed. Report saved to $REPORT."
```

- **Execution**:
  ```bash
  chmod +x manage_expired_accounts.sh
  sudo ./manage_expired_accounts.sh
  ```

- **Explanation**:
  - This script checks all users for expiration status and locks expired accounts.
  - You can automate it to run daily with a cron job.

#### Step 3.3: Set up a Cron Job for Automated Expiration Management

To schedule `manage_expired_accounts.sh` to run daily:

```bash
sudo crontab -e
# Add the following line to run the script every day at midnight
0 0 * * * /path/to/manage_expired_accounts.sh
```

### Step 4: Document Account Management Policies

It’s essential to document policies so that team members understand user account management processes and security expectations.

#### Step 4.1: Create a Documentation Template

Use a template like this for documentation, and store it in a shared drive or version-controlled repository.

#### **Template: Account Management Policy Document**

**1. Purpose**  
This document outlines the user account management policy, including creation, password policy, permissions, and expiration.

**2. Scope**  
Applies to all user accounts on Linux systems managed by the IT team.

**3. User Account Creation**  
- Accounts are created using `create_user.sh` with specified permissions.
- Default shell is `/bin/bash`.
- Accounts are assigned to group `developers` by default.

**4. Password Policy**  
- Passwords must meet complexity requirements: minimum length of 8 characters, must contain uppercase, lowercase, digits.
- Passwords expire every 90 days.

**5. Account Expiration and Removal**  
- Accounts are set to expire 90 days after creation.
- Expired accounts are automatically locked.
- Deletion of accounts follows a manual approval process.

**6. Compliance and Review**  
- Account policies are reviewed quarterly.
- Any issues or incidents are logged and reviewed by the team.


### Summary

1. **Automated User Creation**: Scripted user creation with specified permissions.
2. **Password Policies**: Configured both basic and advanced password policies.
3. **Expired Account Management**: Set up scripts and cron jobs to handle expired accounts.
4. **Documentation**: Created an account management policy document for team reference.


### Final Notes

1. **Testing**: Test each script and configuration change in a development environment.
2. **Security**: Restrict access to user management scripts to administrators only.
3. **Logging**: Add logging to scripts for easier tracking and auditing.
