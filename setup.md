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
That's it! now I can use a terminal from my main computer and ssh to my server.

### (optional) Windows terminal
Since my main computer is a windows (not happy about it, but I'm not ready to have a full linux home yet) my options to SSH are either WSL (meh) Putty (nope, why installing yet another software???) or... simply use powershell since it already have SSH commands.

Problem is the default Powershell is ugly, so I'm using the [Windows Terminal](https://www.microsoft.com/store/productId/9N0DX20HK701), which is a decent, out of the box, multi-language terminal (Go figure why this is not the default CMD and powershell, and why this is a third party software made by Microsoft themselves)

Anyways, In my opinion, the expecience with that one out of the box is much nicer than any of the main solutions (integrated ones, WSL, Git bash etc)

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

### nano
While nano is usually present on linux, it doesn't have code highlights by default.
I'm using a third party library for this, and it can be easily installed using this command:
```bash
curl https://raw.githubusercontent.com/scopatz/nanorc/master/install.sh | sh
```
All done!

### htop
Htop is a basic ressource monitor, but a little bit more fleshed out than top
```bash
sudo apt install htop
```

### (Optional) Fancontrol and lm-sensors
On my previous home server, I used to have a very little and annoying fan running full speed. This utility and sensor suite allowed me to reduce the speed of the fan, and make it "smarter" using the cpu temperature.

Most modern motherboard nowadays have integrated fan speed curves, so I'm not using it anymore, but just in case [here is a stackexchange post on how to install and configure this](https://askubuntu.com/a/46135/1551604)


## 2: Hardware

### Checking your drives (new install / new drives)

- [checking the new drives](https://github.com/Spearfoot/disk-burnin-and-testing)

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

### Setting up mergerfs
MergerFS will... merge multiple disks into one. It is a process easier and faster than using raid, or ZFS
- [link one, from PMV](https://perfectmediaserver.com/installation/manual-install.html)
- [link two](https://forums.serverbuilds.net/t/setting-up-media-server-using-ubuntu-and-snapraid/239)
- [link three](https://zackreed.me/mergerfs-another-good-option-to-pool-your-snapraid-disks/)

Now, my mergerFs is already setup, so here is the line for it, but I'll later explain how to set it up:
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

## 3: Hardening

I'm not a security expert, but I know a few things into security. None of those steps are mandatory, but I would advise to do them at least to understand how it works.

### SSH keys & passwordless remote control
The very first thing is about using SSH keys instead of a password.

This is not needed: By default your network is closed, no one can access your server because the firewall will refuse any connection from outside. The only way to connect to your server is by being connected to your network (eg. Wifi, Ethernet, etc). 

But if you ever want to open it (say remote SSH from outside) then this step is mandatory to prevent brute force attacks (trying every combination of passwords possible until the SSH is accepted).

**Additionally this step will allow you to connect with SSH without having to enter a password, which is very convenient.**

Your first step is to generate a key

```bash
# this commands works both for Bash (linux) and powershell (Windows)
ssh-keygen -t rsa
```

Then send it to the server. There are many methods to do this, I just find the easiest one (that also works with windows) to be using scp.

Make sure before doing this, that the .ssh folder exists on your server
```bash
# Linux/Bash
scp ./ssh/id_rsa.pub user@host:.ssh/authorized_keys

# Powershell
scp  scp .\.ssh\id_rsa.pub username@host:.ssh/authorized_keys


# once done, you can try this by doing 
ssh usernmame@host
# you should be logged in imediately without password
```

Now into securing this! Connect to your remote host and edit the file /etc/ssh/sshd_config with root privileges

Protocol
The protocol 1 is an outdated version that should not be used. By default ubuntu do not allow anymore this, but a quick check will help verifying this
```bash
ssh -1 usernam@host
# this command should return an error. 
# If it doesn't, then you should seriously consider updating your software
```

Now into the other options:
```bash
# uncomment, edit or add the following the the file
PermitEmptyPasswords no
PasswordAuthentication no
PermitRootLogin no
```

Restart sshd, and now your configuration should have been updated
```bash
sudo systemctl restart sshd

# then try to ssh with a password
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no usernamer@server
# you should have a permission denied
```

### Changing SSH port
Now this one is totally optional, if you did the previous steps, and the next one about fail2ban you will be fine. Again this is only in the case of opening your network, if you want to have ssh login only at home, skip this step.

Internet is however full of scripts. So opening your port 22 will generate a lot of logs and traffic (denied of course). To reduce this, just change the port 22 to another one.

### Fail2ban

### Using a Ddns
So this one is not really an option, more of "if you have a static IP, maybe change it", since most internet providers are not giving static IPs.

*please note there that I'm talking about your public IP, **NOT** your local IP. eg Imagine you live in a building, your public ip is the building address, your local ip is your flat number.

*For your local address, you can use statics IP for convenience. Your router DNS will also work tho (by typing in your web address hostname:service port, eg for my home assistant "http://server:8123)*


### Environment variables
Okay so this one is really for peace of mind. If you do not share your configuration all is fine. However is you think about putting this online (github or a tutorial) you should not save database passwords or anything set up by environment variables.
[env variables in docker compose](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)

## 4: Docker-compose
- linuxserver.io
