# VeraCrypt on Manjaro Linux

TL;DR

## Background

- VeraCrypt
- Security and encryption on Linux
- Manjaro Linux

## Entire Volume Encryption vs File Container Encryption

There are two options available to encrypt and decrypt the contents.

## Disk Preparing

- Find out disk device information `lsblk`
- Use `fdisk /dev/sdX`
- File system format
    - `btrfs`

After partitioning the disk, the output of `lsblk` will be

```bash
lsblk
NAME                                 MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                                    8:0    0   1.8T  0 disk  
└─sda1                                 8:1    0   1.8T  0 part  
```

## Command Line

#### Installation
```bash
pacman -S veracrypt
```

#### Configuration

```bash
sudo veracrypt -t --quick -c /dev/sdX
```

#### Mount

```bash
sudo mkdir -p /mnt/sda1 && sudo chown -R USER_GROUP:USER_ID
sudo mount /dev/sda1 /mnt/sda1
```

#### Backup File with `rsync`

```bash
rsync ~/pool/ /mnt/sda1/pool/ --archive -P --delete -v --exclude '.DS_Store' --dry-run
```

- `archive`: keeps file attributes
- `P`: show progress
- `delete`: [be careful] it will delete _everything_ on desitination disk if it doesn't exist on source
- `v`: verbose
- `exclude`: not sync'd
- `dry-run`: test _only_, not committed

#### Unmount

```bash
veracrypt -d /dev/sdX
```