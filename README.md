# Automated Borg Backup with Home Assistant OS

This guide explains how to set up automated backups for Home Assistant OS using **BorgBackup** and a **cron job**. The backups are stored remotely on BorgBase, and old files are pruned automatically. The deployment of the backup script is automated using Ansible.

---

## Prerequisites

- **Home Assistant OS** with Advanced SSH Add-on installed.
- Automatic backups enabled in Home Assistant to local storage.
- A BorgBase repository ready for use.
- Ansible installed on your local machine.

---

## Steps to Set Up

### 1. Set Up the Ansible Repository

1. Clone the repo.

2. Copy the variable template file and update it with your specific settings:
   ```sh
   cp group_vars/all.yml.tpl group_vars/all.yml
   ```

3. Edit `group_vars/all.yml` to set the required variables:
   - `borg_repo`: Your BorgBase repository URL.
   - `borg_passphrase`: Your Borg repository passphrase.
   - `backup_dirs`: The directory to be backed up.

   Example:
   ```yaml
   borg_repo: 'ssh://user@host.repo.borgbase.com/./repo'
   borg_passphrase: 'YOUR-PASSPHRASE'
   backup_dirs: '/backup'
   ```

---

### 2. Run the Ansible Playbook to Deploy the Backup Script

Execute the Ansible playbook to deploy the backup script to your Home Assistant machine:
```sh
ansible-playbook main.yml
```

This will:
- Deploy the backup script `/homeassistant/borg/my-backup.sh` to the target machine.
- Ensure the script has executable permissions (`0755`).

---

### 3. SSH into Home Assistant and Configure Manually

1. Access your Home Assistant machine:
   ```sh
   ssh root@ha-ip
   ```

2. Create the required directories and generate SSH keys for BorgBase:
   ```sh
   mkdir -p /homeassistant/borg/.ssh
   ssh-keygen -t ed25519 -f /homeassistant/borg/.ssh/borgbase-partial
   ```

3. Upload the public key (`/homeassistant/borg/.ssh/borgbase-partial.pub`) to your BorgBase repository.

---

### 4. Configure the Advanced SSH Add-on

1. Install additional packages:
   ```sh
   cronie
   ```

2. Add the cron job for automatic backups:
   ```sh
   crond -f &
   (echo '15 1 * * * /homeassistant/borg/my-backup.sh >> /dev/null 2>&1') | crontab -
   ```

   Choose the time for the cron job to run (`15 1` represents 1:15 AM daily).

3. Save and restart the Advanced SSH Add-on.

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

3. Verify that backups are being uploaded to BorgBase.

---

## Maintenance

- **Logs**: Redirect cron output to a log file for debugging by modifying the cron job:
   ```sh
   15 1 * * * /homeassistant/borg/my-backup.sh >> /homeassistant/borg-backup.log 2>&1
   ```
