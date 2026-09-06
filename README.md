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
   chmod 600 group_vars/all.yml  # cp inherits your umask, which can leave the passphrase world-readable
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
ansible-playbook -i inventory main.yml
```

`-i inventory` is required: the repo directory is world-writable, so Ansible ignores `ansible.cfg` and would otherwise find no hosts. The Advanced SSH add-on has no sftp subsystem, so Ansible's default transfer falls back to its piped mode automatically (with a couple of harmless "sftp/scp transfer mechanism failed" warnings) — `ansible_ssh_transfer_method=piped` in `inventory` skips straight to it.

This will:
- Deploy `/homeassistant/borg/borg-secrets.env` (mode `0600`) holding `BORG_REPO`, `BORG_PASSPHRASE`, `BACKUP_DIRS` from `group_vars/all.yml`.
- Deploy the backup script `/homeassistant/borg/my-backup.sh`, which sources that file — reading the script never prints the passphrase.
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

- **Logs**: The script appends timestamped output to `/homeassistant/borg-backup.log` and trims it to the last 20000 lines after each run. No cron redirect needed.

- **Retry**: A failed borg run is retried once after 15 minutes.

- **Local backup pruning**: runs unconditionally, whether or not the offsite push succeeded — a floor keeps the newest 4 files no matter their age, and a ceiling drops anything beyond that floor once it's older than 8 days. This bounds local disk usage even through a long BorgBase outage, without ever losing the most recent backups.

- **Secrets**: `borg_repo`/`borg_passphrase`/`backup_dirs` from `group_vars/all.yml` are deployed to `/homeassistant/borg/borg-secrets.env` (mode `0600`), sourced by the script rather than embedded in it. `group_vars/all.yml` stays gitignored and local-only — the passphrase is not rotated and does not move into any encrypted store elsewhere, since the Pi must be able to back itself up with no other machine involved.

- **Outcome reporting**: every run reports to Home Assistant, not only to the log — `input_text.borg_backup_last_status` on every run, plus `input_datetime.borg_backup_last_success` and `input_number.borg_backup_last_success_epoch` on success only. It authenticates through the Advanced SSH add-on's own Supervisor token (`http://supervisor/core/api`), so no separate HA credential is deployed here. A failure to reach Home Assistant does not affect the script's own exit code.
