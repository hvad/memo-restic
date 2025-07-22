# README: Local Restic Backups on Fedora

This guide provides a comprehensive walkthrough on setting up local backups 
using **Restic** and automating them with **systemd timers** on Fedora.

-----

## Table of Contents

1.  [Prerequisites](#1-Prerequisites)
2.  [Initialize the Restic Repository](#2-Initialize-the-restic-repository)
3.  [Create the Backup Script](#3-Create-the-backup-script)
4.  [Configure Systemd Unit Files](#-4-Configure-systemd-unit-files)
      * [Restic Backup Service (`.service`)]
      * [Restic Backup Timer (`.timer`)]
5.  [Enable and Start the Systemd Timer](#-5-Enable-and-start-the-systemd-timer)
6.  [Verify Backups and Status](#-6-Verify-backups-and-status)
7.  [Restoring Data](#-7-Restoring-data)
8.  [Backup Policy](#-8-Backup-policy)

-----

## 1 Prerequisites

Before you begin, ensure you have Restic installed and a clear understanding 
of what you want to back up and where your backup repository will reside.

**Install Restic on Fedora:**

Restic is available in the official Fedora repositories.

```bash
sudo dnf install restic
```

Verify the installation:

```bash
restic version
```

-----

## 2 Initialize the Restic Repository

A Restic repository is where your encrypted backup data will be stored.
For local backups, this will be a directory on your filesystem.

1.  **Create a directory for your Restic repository.**
    It's recommended to place this on a separate drive or partition if possible, 
to isolate it from your primary data. For example:

    ```bash
    mkdir -p ~/backups
    ```

    *Replace `~/backups` with your desired path.*

2.  **Initialize the Restic repository.**

    ```bash
    restic init --repo ~/backups
    ```

    You'll be prompted to create a **strong password** for this repository.
This password is critical as it encrypts all your backup data. **Do not lose it\!**

3.  **Store the password securely (optional, but recommended for automation).**
    For automated backups, you'll want Restic to access the password without manual
intervention. Store it in a file with restricted permissions.

    ```bash
    echo "YourSuperSecureResticPassword" > ~/.restic_password
    chmod 600 ~/.restic_password
    ```

    *Replace `"YourSuperSecureResticPassword"` with your actual password.* The `chmod 600`
command ensures only the file owner can read or write to this file.

-----

## 3 Create the Backup Script

1.  **Create a new script file.**

    ```bash
    mkdir -p ~/.local/bin
    vim ~/.local/bin/restic-backup.sh
    ```

2.  **Add the following content to the script.**
    *Adjust the `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE`, `EXCLUDE_FILE`, `LOG_FILE`,
and backup paths (`/home/user/Documents`, etc.) to match setup.*

    ```bash
    #!/bin/bash

    # --- Restic Configuration ---
    export RESTIC_REPOSITORY="/home/user/backups" # Path to Restic repository
    export RESTIC_PASSWORD_FILE="/home/user/.restic_password" # Path to Restic password file

    # --- Backup Paths ---
    # List of directories/files to back up
    BACKUP_PATHS=(
        "/home/user/Documents"
        "/home/user/Pictures"
        "/home/user/Config" # Example: a directory with important config files
        # Add more paths as needed
    )

    # --- Exclude File (Optional) ---
    # Create this file if you want to exclude specific patterns or paths
    # Example content for /home/user/.restic_excludes.txt:
    # *.tmp
    # /home/user/Documents/cache_files
    EXCLUDE_FILE="/home/user/.restic_excludes.txt"

    # --- Logging ---
    LOG_DIR="/var/log/restic"
    LOG_FILE="${LOG_DIR}/backup_$(date +%Y%m%d).log" # Daily log file
    # Or, for a single log file: LOG_FILE="${LOG_DIR}/restic_backup.log"

    # --- Retention Policy ---
    # Keep the last 7 daily snapshots
    # Keep the last 4 weekly snapshots
    # Keep the last 6 monthly snapshots
    # Keep the last 1 yearly snapshot
    RETENTION_POLICY="--keep-daily 7 --keep-weekly 4 --keep-monthly 6 --keep-yearly 1"

    # --- Script Logic ---
    mkdir -p "${LOG_DIR}" # Ensure log directory exists

    echo "--- Restic Backup Started: $(date) ---" >> "${LOG_FILE}" 2>&1

    # Perform the backup
    if [ -f "${EXCLUDE_FILE}" ]; then
        restic backup "${BACKUP_PATHS[@]}" --tag "daily-backup" --exclude-file "${EXCLUDE_FILE}" >> "${LOG_FILE}" 2>&1
    else
        restic backup "${BACKUP_PATHS[@]}" --tag "daily-backup" >> "${LOG_FILE}" 2>&1
    fi

    BACKUP_EXIT_CODE=$?
    if [ $BACKUP_EXIT_CODE -eq 0 ]; then
        echo "Restic backup completed successfully." >> "${LOG_FILE}" 2>&1
    else
        echo "Restic backup failed with exit code: $BACKUP_EXIT_CODE." >> "${LOG_FILE}" 2>&1
    fi

    echo "--- Restic Prune Started: $(date) ---" >> "${LOG_FILE}" 2>&1

    # Apply the retention policy and prune the repository
    restic forget ${RETENTION_POLICY} --prune >> "${LOG_FILE}" 2>&1
    PRUNE_EXIT_CODE=$?
    if [ $PRUNE_EXIT_CODE -eq 0 ]; then
        echo "Restic prune completed successfully." >> "${LOG_FILE}" 2>&1
    else
        echo "Restic prune failed with exit code: $PRUNE_EXIT_CODE." >> "${LOG_FILE}" 2>&1
    fi

    echo "--- Restic Backup Finished: $(date) ---" >> "${LOG_FILE}" 2>&1
    ```

3.  **Make the script executable.**

    ```bash
    chmod +x ~/.local/bin/restic-backup.sh
    ```

4.  **Create the optional exclude file.**

    ```bash
    vim ~/.restic_excludes.txt
    ```

    Add patterns for files/directories you want to exclude, for example:

    ```
    *.log
    *.tmp
    */cache/*
    /home/user/Downloads
    ```

-----

## 4 Configure Systemd Unit Files

Systemd timers require two files: a `.service` unit that defines the task to be executed,
and a `.timer` unit that defines when and how often the service should run. 
Create these as *user services* so they run under user account, not as root.

1.  **Create the systemd user service directory.**

    ```bash
    mkdir -p ~/.config/systemd/user/
    ```

### Restic Backup Service (`.service`)

This file tells systemd how to run your backup script.

1.  **Create the service file:**

    ```bash
    vim ~/.config/systemd/user/restic-backup.service
    ```

2.  **Add the following content:**

    ```ini
    [Unit]
    Description=Restic Local Backup Service
    Documentation=https://restic.readthedocs.io/en/latest/
    After=network-online.target

    [Service]
    Type=oneshot
    ExecStart=/home/user/.local/bin/restic-backup.sh
    # If you want to run this service only when logged in, remove the following line.
    # Otherwise, it will run even if you're not logged into a desktop session.
    # For automated backups, it's usually desired to run without an active session.
    User=user # Specify your username here
    Group=user # Specify your group here (usually same as username)
    Environment="HOME=/home/user" # Ensure HOME is set correctly for ~ in script paths
    StandardOutput=append:/var/log/restic/restic-service.log
    StandardError=append:/var/log/restic/restic-service.log
    # Consider setting a timeout if backups might hang
    # TimeoutStartSec=3600
    ```

    *Replace `user` with your actual username.*

### Restic Backup Timer (`.timer`)

This file defines the schedule for your backup service.

1.  **Create the timer file:**

    ```bash
    vim ~/.config/systemd/user/restic-backup.timer
    ```

2.  **Add the following content:**
    This example sets the timer to run daily at 02:00 AM.

    ```ini
    [Unit]
    Description=Run Restic Local Backup Daily at 02:00 AM

    [Timer]
    # Define when the timer should run
    # OnCalendar= expresses a calendar event specification
    # "daily" is a shortcut for "*-*-* 00:00:00"
    # "daily" can also be used with an offset, e.g., OnCalendar=daily 02:00:00
    OnCalendar=*-*-* 02:00:00
    # Optional: If you want to randomize the start time slightly to avoid stampedes on network resources
    # RandomSec=30m
    # Optional: Delay start after boot, if needed (e.g., for network availability)
    # OnBootSec=15min
    # Optional: Ensure the service runs if the system was off during the scheduled time
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```

    *Adjust `OnCalendar` to suit your backup frequency. Some common examples:*

      * `OnCalendar=daily`: Once a day, at midnight.
      * `OnCalendar=weekly`: Once a week, on Monday at midnight.
      * `OnCalendar=Sat *-*-* 03:00:00`: Every Saturday at 03:00 AM.
      * `OnCalendar=minutely`: Every minute (for testing only, highly verbose).

-----

## 5 Enable and Start the Systemd Timer

Once the unit files are created, you need to tell systemd to load them and enable the timer.

1.  **Reload the systemd user manager to pick up the new files:**

    ```bash
    systemctl --user daemon-reload
    ```

2.  **Enable the timer.**
    This makes the timer start automatically on login or system boot if user services are enabled globally.

    ```bash
    systemctl --user enable restic-backup.timer
    ```

3.  **Start the timer immediately (optional, if you don't want to wait for the scheduled time):**

    ```bash
    systemctl --user start restic-backup.timer
    ```

-----

## 6 Verify Backups and Status

You can check the status of your timer and service, and list your Restic snapshots.

1.  **Check the status of your timer:**

    ```bash
    systemctl --user status restic-backup.timer
    ```

    Look for `Active: active (waiting)` and `Next: <timestamp>`.

2.  **Check the status of your service (after it has run):**

    ```bash
    systemctl --user status restic-backup.service
    ```

    This will show if the last run was successful.

3.  **View the service logs:**

    ```bash
    journalctl --user -u restic-backup.service
    ```

    You can also check the `LOG_FILE` you defined in your `restic-backup.sh` script (e.g., `/var/log/restic/backup_YYYYMMDD.log`).

4.  **List Restic snapshots to confirm backups are being created:**

    ```bash
    restic snapshots --repo ~/backups --password-file ~/.restic_password
    ```

    You should see new snapshots appearing after each successful run of the timer.

-----

## 7 Restoring Data

While this guide focuses on setup, remember the ultimate goal is data recovery.

To restore data:

1.  **List snapshots:**

    ```bash
    restic snapshots --repo ~/backups --password-file ~/.restic_password
    ```

    Note the **snapshot ID** you want to restore.

2.  **Restore a full snapshot:**

    ```bash
    restic restore <snapshot_ID> --target /path/to/restore/destination --repo ~/backups --password-file ~/.restic_password
    ```

3.  **Restore specific files/directories:**

    ```bash
    restic restore <snapshot_ID> --target /path/to/restore/destination --include /path/in/backup/file.txt --repo ~/backups --password-file ~/.restic_password
    ```

    Ensure the `target` directory is empty or won't overwrite important files.

For more detailed restoration options, refer to the [Restic documentation](https://www.google.com/search?q=https://restic.readthedocs.io/en/latest/050_restoring.html).

-----

## 8 Backup Policy

The `restic forget` command, included in your `restic-backup.sh` script, implements your retention policy.

The example policy is:

  * `--keep-daily 7`: Retain the last 7 snapshots if there's at least one for each day.
  * `--keep-weekly 4`: Retain the last 4 snapshots if there's at least one for each week.
  * `--keep-monthly 6`: Retain the last 6 snapshots if there's at least one for each month.
  * `--keep-yearly 1`: Retain the last 1 snapshot if there's at least one for each year.
  * `--prune`: This command physically removes the data chunks from the repository that are no longer referenced by any remaining snapshots, freeing up disk space.

Adjust these parameters in your `restic-backup.sh` script (`RETENTION_POLICY` variable) to fit your specific needs regarding data history and storage space. You can test a policy without making changes using `--dry-run`:

```bash
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --keep-yearly 1 --prune --dry-run --repo ~/backups --password-file ~/.restic_password
```
