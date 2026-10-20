# Migrating data off a Synology NAS

[Version française](README.md)

Copy your data off a Synology before selling or replacing it.
Two verified copies: one on an external (USB) disk, one on a disk freed inside the NAS.
Both in **plain ext4**: any Linux can read them, without DSM or mdadm.

Tested on a DS1621+ running DSM 7. Commands are typed over SSH, on the NAS.

> **Golden rule: erase nothing until two copies are verified.**
>
> **Before you start**, if you can: one more backup, in the cloud (Hyper Backup to Synology C2, a Storage Box,
> an S3 service…). It also protects you from a mistake during the migration.

## 1. Free a disk

Clean up, so the data fits on a single volume:

- Uninstall the apps you no longer use (**Package Center**).
- Delete the shared folders you no longer need (**Control Panel → Shared Folder**).
- Gather the others on one volume: **Shared Folder → Edit → Location**.
- Then, in **Storage Manager**: delete the volume and the storage pool of the disk to free.
- Never touch the pool that holds the data.

## 2. Plug in the external disk

- Use a USB port **at the back**. The front ones may run at USB 2: about 30 MB/s instead of 300.
- Check the speed: `SuperSpeed` = USB 3, `high-speed` = USB 2.

  ```sh
  dmesg | grep -i -E 'usb.*(high-speed|superspeed)' | tail -3
  ```

- DSM mounts it under `/volumeUSB1/usbshare`. Check its format:

  ```sh
  mount | grep volumeUSB
  ```

  ext4 is best. NTFS and exFAT keep neither owners, permissions nor hard links.

- Format it as ext4 (this erases it): **Control Panel → External Devices**, select the disk, **Format**,
  **Entire disk**, **ext4**. DSM creates a plain ext4 partition, readable on any Linux.

## 3. Enable SSH

- DSM: **Control Panel → Terminal & SNMP → Enable SSH service**.
- From your PC:

  ```sh
  ssh admin_account@nas_address
  sudo su -                         # become root, with the account's password
  ```

- Run the copies in `screen` or `tmux` if available, or with `nohup`.
  A dropped SSH session would stop the copy. Running the same command again resumes it.

## 4. Copy to the external disk

Pick the folders to keep: `ls -la /volume1`.
Folders starting with `@` or `#` are DSM internals: do not copy them.

```sh
rsync -aH --info=progress2 --exclude='@eaDir' --exclude='#recycle' --exclude='#snapshot' \
  /volume1/homes/my_account /volumeUSB1/usbshare/
```

Without a `/` after the source folder, rsync creates `my_account/` in the destination.

Verify. Nothing should be printed:

```sh
rsync -aHc --dry-run --itemize-changes --exclude='@eaDir' --exclude='#recycle' --exclude='#snapshot' \
  /volume1/homes/my_account /volumeUSB1/usbshare/
```

NTFS or exFAT disk: replace `-aHc` with `-rc`. Only the content is compared then.

## 5. Prepare the freed disk

DSM keeps a copy of its system (`md0`) and swap (`md1`) on **every** disk.
Take the disk out of them before repartitioning it. Otherwise DSM will want to "repair" it.

**Identify the disk.** Disks are named `/dev/sata1`, `/dev/sata2`…

```sh
cat /proc/mdstat
smartctl -i /dev/sataX | grep -i -E 'serial|capacity'
```

- The data lives in `md2` (or higher): its disks are off-limits.
- The freed disk only shows up in `md0` and `md1`.
- Write down its serial number. Read it again before every command that writes.

**Take it out of DSM's system and swap.** They stay on the other disks.

```sh
mdadm /dev/md0 --fail /dev/sataXp1 --remove /dev/sataXp1
mdadm /dev/md1 --fail /dev/sataXp2 --remove /dev/sataXp2
cat /proc/mdstat                                   # sataX is gone; the data is untouched
```

**Erase its signatures.** DSM has no `wipefs`; `mdadm` is enough.

```sh
mdadm --zero-superblock /dev/sataXp1 /dev/sataXp2
mdadm --examine /dev/sataXp1 /dev/sataXp2          # No md superblock detected
```

**One partition, in ext4.** The only step with no way back: read the serial number again.

```sh
parted -s /dev/sataX mklabel gpt mkpart backup ext4 0% 100%
mkfs.ext4 -L backup /dev/sataX1
mkdir -p /mnt/backup && mount /dev/sataX1 /mnt/backup
df -h /mnt/backup
```

DSM will show this disk as "not initialized", and `md0`, `md1` as "degraded".
That is expected: click neither **Repair** nor **Create**.

## 6. Copy to that disk

Same commands as in step 4, to `/mnt/backup/`: the copy, then the check.

Ballpark on a DS1621+: 126 MB/s disk to disk, 31 MB/s to an SSD plugged in at USB 2.

## 7. Before shutting the NAS down

- `umount /mnt/backup`.
- Remove the SSD cache, if any (**Storage Manager → SSD Cache**).
- Export the virtual machines to keep (**Virtual Machine Manager → Export**).
- Unlink the NAS from the Synology account, and from QuickConnect.
- Never put the copy disk back into a Synology: DSM may offer to initialize it, which erases it.
- No need to reset the NAS: just remove all the drives. DSM and its settings live on the drives (the `md0` system
  partition), not in the NAS, and cannot be installed on M.2 NVMe drives ([Synology: install
  DSM](https://kb.synology.com/en-global/DSM/tutorial/How_to_install_DSM)). With no drive, the NAS shows up as
  "Not installed", like a new one. A reset from DSM, on the other hand, erases the drives inside.

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — pazpop
