# Automated Borg Backup with Home Assistant OS

This guide explains how to set up automated backups for Home Assistant OS using **BorgBackup** and a **cron job**. The backups are stored remotely on BorgBase and old files are pruned automatically.

---

## Prerequisites

- **Home Assistant OS** with Advanced SSH Add-on installed.
- Automatic backups enabled in Home Assistant to local storage. 
- A BorgBase repository ready for use.

---

## Steps to Set Up

### 1. SSH into Home Assistant
Access your Home Assistant machine:
```sh
ssh root@ha-ip
```

---

### 2. Create Directories and Generate SSH Keys
Prepare the directory structure and generate an SSH key for BorgBase:
```sh
cd /homeassistant
mkdir -p borg/.ssh
ssh-keygen -t ed25519 -f borg/.ssh/borgbase-partial
```

Upload the public key (`borg/.ssh/borgbase-partial.pub`) to your BorgBase repository.

---

### 3. Create the Backup Script
Create a backup script:
```sh
nano /homeassistant/borg/my-backup.sh
```

Add the following content:
- replace `BORG_PASSPHRASE` with your Borg passphrase.
- replace `BORG_REPO` with your Borg repository.

```sh
#!/bin/sh
/usr/local/bin/docker pull alpine:latest
/usr/local/bin/docker run \
  -e BORG_REPO="ssh://user@host.repo.borgbase.com/./repo" \
  -e BORG_RSH="ssh -i /root/.ssh/borgbase-partial" \
  -e BORG_PASSPHRASE=YOUR-PHRASE \
  -e BACKUP_DIRS=/backup \
  -v /mnt/data/supervisor/homeassistant/borg:/root \
  -v /mnt/data/supervisor/backup:/backup:ro \
  --cap-add SYS_ADMIN --device /dev/fuse \
  --security-opt apparmor:unconfined \
  --rm --name borg-backup \
  alpine:latest sh -c 'apk add --no-cache borgbackup openssh && borg create --stats --progress "${BORG_REPO}::$(date +%Y-%m-%d-%H-%M)" $BACKUP_DIRS && borg prune --list "${BORG_REPO}" --keep-daily=7 --keep-weekly=4 --keep-monthly=6'
/usr/bin/find /backup -type f -mtime +7 -exec /bin/rm -f {} \;
```

**Old Backups**: Automatically deletes files in `/backup` older than 7 days.


Make the script executable:
```sh
chmod +x /homeassistant/borg/my-backup.sh
```

---

### 4. Configure the Advanced SSH Add-on

Additional packages:
```sh
cronie
```

Additional init commands:
```sh
crond -f &

(echo '15 1 * * * /homeassistant/borg/my-backup.sh >> /dev/null 2>&1') | crontab -
```
Choose the time for the cron job to run.

Save and restart the SSH add-on.

---

### 5. Verify and Test
1. Test the backup script manually:
   ```sh
   /homeassistant/borg/my-backup.sh
   ```

2. Confirm the cron job is running:
   ```sh
   crontab -l
   ```

3. Check BorgBase to verify that backups are being uploaded.

---

## Maintenance
- **Logs**: Redirect cron output to a log file for debugging:
  ```sh
  15 1 * * * /homeassistant/borg/my-backup.sh >> /homeassistant/borg-backup.log 2>&1
  ```

---

That's it! Your Home Assistant setup is now configured to back up automatically to BorgBase. 🎉