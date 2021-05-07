# setting up the server

## 1: Software

- [Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- [Docker-compose](https://docs.docker.com/compose/install/#install-compose)
- [docker-compose autocompletion](https://docs.docker.com/compose/completion/#bash)
- backup?
- [mergerfs](https://github.com/trapexit/mergerfs) for a pool of disks

## 2: Hardware
- automatic mount of volumes (hard drives & deconz)
- [checking the new drives](https://github.com/Spearfoot/disk-burnin-and-testing)

### Setting up mergerfs
MergerFS will... merge multiple disks into one. It is a process easier and faster than using raid, or ZFS
- [link one, from PMV](https://perfectmediaserver.com/installation/manual-install)
- [link two](https://forums.serverbuilds.net/t/setting-up-media-server-using-ubuntu-and-snapraid/239)
- [link three](https://zackreed.me/mergerfs-another-good-option-to-pool-your-snapraid-disks/)

## ?: Hardening
- [env variables in docker compose](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)
