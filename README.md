# Automated Backup & Disaster Recovery Engine (Bash & Cron)

## 📌 Project Overview
In production server environments, unmanaged backups lead to disk exhaustion, while missing backups risk permanent data loss. 

This project implements an automated, production-grade **Backup and Disaster Recovery Engine** on **AWS EC2 (Ubuntu Linux)** using native Bash scripting and Linux scheduling primitives:
* **Targeted Directory Compression:** Archives critical application assets (`/var/www/html`) and web proxy configurations (`/etc/nginx`) into timestamped `.tar.gz` archives.
* **Automated Retention Management:** Prevents disk saturation by pruning backups older than 7 days using the `find` utility.
* **Structured Auditing & Logging:** Logs backup lifecycle events, file sizes, and status codes to `/var/log/backup.log`.
* **Zero-Touch Automation:** Scheduled via `cron` to run nightly at 2:00 AM off-peak hours.
* **Disaster Recovery Validation:** Verified operational integrity through simulated file loss and live restore drills.

---

## 🏗️ Architecture & Workflow

```text
[ Cron Daemon (Nightly @ 02:00) ]
              |
              v
[ Bash Engine: /usr/local/bin/backup_manager.sh ]
              |
     +--------+--------+------------------------+
     |                 |                        |
     v                 v                        v
[ Create Archive ] [ Apply Retention ]   [ Write Logs ]
tar -czf           find -mtime +7 -delete >> /var/log/backup.log
     |                 |
     v                 v
[ /var/backups/ ]   [ Storage Clean ]
```

---

## 📜 Shell Script Source (`backup_manager.sh`)

```bash
#!/bin/bash

# ==============================================================================
# Script Name: backup_manager.sh
# Purpose    : Automated Directory Backup with Retention & Logging
# Author     : Linux System Administrator
# ==============================================================================

# Configurations
BACKUP_DIR="/var/backups/system_backups"
SOURCE_DIRS=("/var/www/html" "/etc/nginx")
LOG_FILE="/var/log/backup.log"
RETENTION_DAYS=7
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
BACKUP_ARCHIVE="${BACKUP_DIR}/system_backup_${TIMESTAMP}.tar.gz"

# Logging function
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | sudo tee -a "$LOG_FILE" > /dev/null
}

log_message "=== Starting Automated Backup Routine ==="

# Check if Backup Directory exists
if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
    log_message "Created missing backup directory: $BACKUP_DIR"
fi

# Create Compressed Archive
log_message "Archiving source paths: ${SOURCE_DIRS[*]}"
tar -czf "$BACKUP_ARCHIVE" "${SOURCE_DIRS[@]}" 2>> "$LOG_FILE"

# Check exit status of tar command
if [ $? -eq 0 ]; then
    ARCHIVE_SIZE=$(ls -lh "$BACKUP_ARCHIVE" | awk '{print $5}')
    log_message "SUCCESS: Archive created at $BACKUP_ARCHIVE (Size: $ARCHIVE_SIZE)"
else
    log_message "ERROR: Backup archive creation failed!"
    exit 1
fi

# Retention Cleanup: Delete archives older than RETENTION_DAYS
log_message "Running retention policy: Deleting backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -type f -name "system_backup_*.tar.gz" -mtime +"$RETENTION_DAYS" -exec rm -f {} \; >> "$LOG_FILE" 2>&1
log_message "Retention cleanup completed."

log_message "=== Backup Routine Completed Successfully ==="
exit 0
```

---

## 🛠️ Step-by-Step Implementation & Deployment

### 1. Storage & Audit Setup
Initialize dedicated backup storage and allocate write permissions for the log sink:
```bash
sudo mkdir -p /var/backups/system_backups
sudo touch /var/log/backup.log
sudo chmod 644 /var/log/backup.log
```

### 2. Script Installation & Execution Rights
Install the script into standard system binary path and grant executable flags:
```bash
sudo chmod +x /usr/local/bin/backup_manager.sh
sudo /usr/local/bin/backup_manager.sh
```

### 3. Nightly Schedule Configuration (Cron)
Configure root crontab to execute the maintenance job daily at 2:00 AM:
```bash
sudo crontab -e
```
*Appended configuration:*
```cron
0 2 * * * /usr/local/bin/backup_manager.sh > /dev/null 2>&1
```

---

## 🔍 Disaster Recovery & Validation Drill

A backup system is only as reliable as its restoration process. The workflow was verified through an active drill:

### Step 1: Simulate Unintended Data Deletion
```bash
sudo rm /var/www/html/index.html
# Accessing web endpoint yields 403 Forbidden / Empty response
```

### Step 2: Extract & Restore from Point-in-Time Archive
```bash
# Locate most recent backup archive
ls -lh /var/backups/system_backups/

# Extract archive directly into root filesystem preserving paths
sudo tar -xzf /var/backups/system_backups/system_backup_*.tar.gz -C /
```

### Step 3: Verification
```bash
# Verify file restoration and integrity
ls -l /var/www/html/index.html
curl -I [http://127.0.0.1:8080](http://127.0.0.1:8080)
# HTTP response returns 200 OK with intact backend data
```

---

## 🚀 Key Takeaways & Sysadmin Skills Demonstrated
* **Linux Shell Scripting:** Robust exit status evaluation (`$?`), dynamic timestamps, array handling, and custom standard output logging functions.
* **Archive & Compression:** Efficient storage usage leveraging `tar -czf` utility.
* **Storage Lifecycle (Retention):** Automated space reclamation utilizing `find` with time-based flags (`-mtime`).
* **Job Scheduling:** System automation and daemon configuration via standard `cron` tables.
* **Disaster Recovery (DR):** End-to-end recovery validation, ensuring Recovery Time Objective (RTO) and Recovery Point Objective (RPO) readiness.
