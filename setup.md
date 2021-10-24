# setting up the server

## 1: Software

- [Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Docker-compose](https://docs.docker.com/compose/install/#install-compose)
- [docker-compose autocompletion](https://docs.docker.com/compose/completion/#bash)
- backup?
- [mergerfs](https://github.com/trapexit/mergerfs) for a pool of disks

## 2: Hardware
- automatic mount of volumes (hard drives & deconz)

The first step is to discover what is connected to the machine and what are the name of the hard drives
```bash
lsblk
```

We can now proceed by identifying the partition standard, create the partition, and format it
```bash
sudo parted /dev/sdX mklabel gpt
sudo parted -a opt /dev/sdX mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L mylabelname /dev/sdX1
```

It's now time to mount this to the filesystem, and to add it to the mergerfs pool

Important note: 
The disk labels are not consistant between reboots, this means in the sdX, the X may change after a reboot, this is why it is important in the configuration to use UUIDs, that will not change
```bash
lsblk -o NAME,LABEL,UUID,MOUNTPOINT
# take note / Copy the UUID, as we will need it on the next step
sudo nano /etc/fstab
```
The fstab can be done using either the label or the UUID, and remember to use a path ending with numbers (ex: disk1, disk2) as this will ease the work later.
```bash
# <file system>    <mount point>      <type>  <options> <dump><pass>
UUID=youruuid     /my/path/to/mount    ext4   defaults     0    0
# Or
LABEL=yourlabel   /path                ext4   defaults     0    0
# Note that the options should be different for an SSD
```
It's time now to mount the drive!
```bash
# if the selected path does not exist yet, you need to create it
sudo mkdir /mbt/disk3
# then we can mount the new disk
sudo mount -a
# Wee check
lsblk --fs
```

Now, my mergerFs is already setup, so here is the line for it, but I'll go later on how to set it up:
```bash
/mnt/disk* /mnt/storage fuse.mergerfs defaults,nonempty,allow_other,use_ino,cache.files=off,moveonenospc=true,dropcacheonclose=true,category.create=mfs,minfreespace=60G,fsname=mergerFS 0 0
```
- [checking the new drives](https://github.com/Spearfoot/disk-burnin-and-testing)

### Setting up mergerfs
MergerFS will... merge multiple disks into one. It is a process easier and faster than using raid, or ZFS
- [link one, from PMV](https://perfectmediaserver.com/installation/manual-install)
- [link two](https://forums.serverbuilds.net/t/setting-up-media-server-using-ubuntu-and-snapraid/239)
- [link three](https://zackreed.me/mergerfs-another-good-option-to-pool-your-snapraid-disks/)

## ?: Hardening
- [env variables in docker compose](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)
