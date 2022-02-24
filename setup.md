# setting up the server

## 1: Software

- [Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Docker-compose](https://docs.docker.com/compose/install/#install-compose)
- [docker-compose autocompletion](https://docs.docker.com/compose/completion/#bash)
- backup?
- [mergerfs](https://github.com/trapexit/mergerfs) for a pool of disks

### SSH
The first step is to install all the software needed to make the server work.
Since I'm beginning a fresh install on a desktop version of ubuntu, I will need OpenSSH, to allow me remote control of the server.
```bash
sudo apt update
sudo apt install openssh-server
```
That's it! now I can use a terminal from my main computer (Powershell, cmd, bash, etc) and ssh to my server.

### Docker & docker-compose
All of the services I'm using on the home server are orchestrated by stacks of docker-compose. To do this, I need to install docker (to handle the containers) and docker-compose to nicely interact with them.
Those commands are directly taken from [Docker's website](https://docs.docker.com/engine/install/ubuntu/)
```bash
sudo apt-get install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# verify the software is working
sudo docker -v
# or
sudo systemctl status docker
```
For convenience, I'm also going to add my user to the docker user group. This will allow me to call docker commands directly, without using sudo
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
# and try it out, this command should not have the permission denied error
docker run hello-world
```
Finally, I want docker to auto-start on boot
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

Now into docker-compose!
```bash
mkdir -p ~/.docker/cli-plugins/
# be careful to check the release version here first: https://github.com/docker/compose/releases
# then update this command accordingly
curl -SL https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
# finally test it
docker-compose version

# if the path did not update, use this command to add it
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# if this still doesn't work, install the ubuntu package, it's a bit behing for updates, but it works
```

### MergerFS
This software will allow to pool hard drives. Aka having HDD1 and HDD2 be seen by the file system as HDD, essentially making a giant drive, and allowing to upgrade the hardware easily.
To install it:
```bash
# Be careful to update version number, and to ensure the release is for the correct OS/hardware
wget https://github.com/trapexit/mergerfs/releases/download/2.33.3/mergerfs_2.33.3.ubuntu-focal_amd64.deb
sudo dpkg -i mergerfs_2.33.3.ubuntu-focal_amd64.deb

# verify it is installed
apt list mergerfs

# finally clean your home folder
rm -r mergerfs_2.33.3.ubuntu-focal_amd64.deb
```




## 2: Hardware

### Identify your hardware and mounting it
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


### Setting up deConz
deConz is the way I can interact with my zigbee network (lights, sensors, etc). To let docker interact with physical parts, I need to set up a few things prior to launching HA and the deconz container

[from their website](https://phoscon.de/en/conbee/install#docker) and the [github repo](https://github.com/deconz-community/deconz-docker)

```bash
# this command add the user to the dialout group
# this group allows access to serial devices
sudo gpasswd -a $USER dialout
```

That's it! Now the only thing left is to bring up the containers


- [checking the new drives](https://github.com/Spearfoot/disk-burnin-and-testing)

### Setting up mergerfs
MergerFS will... merge multiple disks into one. It is a process easier and faster than using raid, or ZFS
- [link one, from PMV](https://perfectmediaserver.com/installation/manual-install.html)
- [link two](https://forums.serverbuilds.net/t/setting-up-media-server-using-ubuntu-and-snapraid/239)
- [link three](https://zackreed.me/mergerfs-another-good-option-to-pool-your-snapraid-disks/)

## ?: Hardening
- [env variables in docker compose](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)
