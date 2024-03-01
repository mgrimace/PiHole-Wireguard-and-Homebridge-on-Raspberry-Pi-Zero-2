# Create an image of the SD card and save it on a network share

we will be using image utilities, found at the following link: https://forums.raspberrypi.com/viewtopic.php?t=332000

## Create a network share

First create an SMB/CIFS share called 'backup'

## SSH to the Pi

1. install `sudo apt-get install cifs-utils`

2. Add your credentials (i.e., your SMB user and password) to `nano ~/.storage_credentials`

3. ```bash
   username=[username] #type it in as-is, without the []
   password=[your password] #type it in as-is, without the []
   ```

4. `chmod 0600 ~/.storage_credentials` so only the owner can read it

5. create your local backup director by `cd to /etc/` and `sudo mkdir backup`

6. `nano /etc/fstab` to mount your smb:

7. ```bash
   #backups
   //[IP]/backup /home/[username]/backup cifs credentials=/home/[username]/.storage_credentials,x-systemd.automount,iocharset=utf8,rw,uid=1000,gid=1000,vers=3.0 0 0
   
   #replace your [IP] and [username] in the entry above
   ```


## Add Image File Utilities

1. `mkdir ~/bin`
2. Extract the image file utilities from the first post [here](https://forums.raspberrypi.com/viewtopic.php?t=332000) into that /home/[user]/bin and make all items executable

3. `cd /image-utils`

## Make a backup

1. `sudo ./image-backup`
2. enter the name  ``/mnt/backup/pi_backup.img`, then hit enter then `y` when prompted to start the backup. Wait patiently.


## Setup scheduled backups

1. `sudo nano backups` to create a backup script, add:

2. ```bash
   #!/bin/bash
   
   BUFPATH="/mnt/backup/"
   BUFNAME="pi_backup_"
   BINPATH="/etc/local/bin/image-backup"
   
   CURDATE="$(date +%F)"
   BUSDATE="$(sed -n 's|^\(\S\+-\S\+\)-.*$|\1|p' <<< ${CURDATE})"
   CURFILE="$(ls ${BUFPATH}${BUFNAME}${BUSDATE}-??.img 2>/dev/null | sed -n "s|^\(${BUFPATH}${BUFNAME}${BUSDATE}-\S\+\.img$\)|\1|p")"
   if [ "${CURFILE}" = "" ]; then
     sudo "${BINPATH}" -i "${BUFPATH}${BUFNAME}${CURDATE}.img,,100"
   else
     sudo "${BINPATH}" "${CURFILE}"
   fi
   ```

3. `crontab -e`, then add:

4. ```bash
   0 4 * * * /home/[user]/backup
   ```


