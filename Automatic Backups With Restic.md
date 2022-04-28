# Automatic Backups With Restic

Backups should be automatic - fire and forget. This guide shows you how it is done!

First of - what is restic? It’s a relatively new backup program for Linux and Windows. You can get your copy at the [release page](https://github.com/restic/restic/releases) or in your repository.

## Act 1: The setup

The way restic works we need a “remote” repository. Bare a little with me before typing anything in your console jockey. We will automate this stuff in a moment.

So you can initialize the repo by issuing the following command: But did you know that restic also reads the environment variable **RESTIC_REPOSITORY**?`restic init --repo /src/backup`

There is also **RESTIC_PASSWORD_FILE** or **RESTIC_PASSWORD** if you like your password better hanging around in the ENV.

Here is what we will do - we will create a file called e.g. in `restic_env``/etc/backup/`

**`/etc/backup/restic_env:`**

    export RESTIC_REPOSITORY="/path/to/repo" 
    export RESTIC_PASSWORD_FILE="/etc/restic/pw.txt"

For this to work we need some file that will source this in our current env; this will be our backup script:

**`/etc/backup/auto_backup.sh:`**

    #!/usr/bin/env bash
    # This script is intended to be run by a systemd timer
    
    # Exit on failure or pipefail
    set -e -o pipefail
    
    #Set this to any location you like
    BACKUP_PATHS = "/home"
    
    BACKUP_TAG=systemd.timer
    
    # How many backups to keep.
    RETENTION_DAYS=14
    RETENTION_WEEKS=16
    RETENTION_MONTHS=18
    RETENTION_YEARS=3
    
    source /etc/backup/restic_env
    
    # Remove locks in case other stale processes kept them in
    restic unlock &
    wait $!
    
    #Do the backup
    
    restic backup \
           --verbose
           --one-file-system \
           --tag $BACKUP_TAG \
           $BACKUP_PATH &
    
    wait $!
    
    # Remove old Backups
    
    restic forget \
           --verbose \
           --tag $BACKUP_TAG \
           --prune \
           --keep-daily $RETENTION_DAYS \
           --keep-weekly $RETENTION_WEEKS \
           --keep-monthly $RETENTION_MONTHS \
           --keep-yearly $RETENTION_YEARS &
    wait $!
    
    # Check if everything is fine
    restic check &
    wait $!
    
    echo "Backup done!"

It’s prudent to run your first backup with this command manually so and give it a try.`chmod +x`

## Act 2: The Automation

Systemd needs a timer and a unit to fire up our little script like cron did in the golden days - and hey if you are still an old school operator using chron just skip ahead; you know what you are doing - for everybody else here is the timer:

**`/etc/systemd/system/backup.timer:`**

    [Unit]
    Description=Backup on schedule
    
    [Timer]
    OnCalendar=daily
    Persistent=true
    
    [Install]
    WantedBy=timers.target

And the service file:

**`/etc/systemd/system/backup.service:`**

    [Unit]
    Description=Backup with restic
    [Service]
    Type=simple
    Nice=10
    ExecStart=/etc/backup/restic_backup.sh
    #$HOME must be set for restic to find /root/.cache/restic/
    Environment="HOME=/root"

## Act 3 : The Execution

Now all is set and done - to activate you only have to run this one command: `systemctl enable backup.timer --now`

You can now use to the schedule and check on the service using with you can even start it manually.`systemctl list-timers | grep backup``systemctl status backup``systemctl start backup`

To follow up on your backups just use `journalctl -u backup.service`
