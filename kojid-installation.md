Installation of Koji Daemon Builder
===================================

I am using Amazon EC2 instance (large). Create new instance in the same
datacenter as Koji Hub with all ports opened. I used RHEL 6.4 64bit system and
attached one ephemeral volume (it must be manually mounted - as /mnt/tmp).

    echo -e "p\nn\np\n1\n\n\np\nw\n" | fdisk /dev/xvdf
    mkfs.ext4 /dev/xvdf1
    mkdir /mnt/tmp
    mount /dev/xvdf1 /mnt/tmp

Since the storage is ephemeral, we don't need to deal with fstab.

Copy our public ssh keys into /root/.ssh/authorizued_keys deleting the entry
there which disables root account.

Also add this entry to the /etc/hosts

    echo "10.42.135.135 kojihub" >> /etc/hosts

Prerequisites
-------------

We assume that two directories are exported on the hub

    mkdir -p /exports/koji /exports/external-repos
    mount /mnt/koji /exports/koji -o bind
    mount /mnt/tmp/external-repos/ /exports/external-repos -o bind

And we assume NFS ports are set and iptables rules too, see
http://lukas.zapletalovi.com/2011/05/export-for-both-nfs-v4-and-v3-clients.html

Make sure the builder is in the same security group as the hub (which is
"default"). To check NFS is working just do this:

    showmount -e kojihub
    Export list for kojihub:
    /exports/external-repos *
    /exports/koji           *
    /exports                *

Installation
------------

Enable EPEL

    yum -y install http://mirror.slu.cz/epel/6/i386/epel-release-6-8.noarch.rpm
    yum -y install koji-builder

Configuration
-------------

First of all, we need to mount several directories from the hub

    mkdir /mnt/koji
    mount -t nfs -o vers=4 kojihub:/koji /mnt/koji
    mkdir /mnt/tmp/external-repos
    mount -t nfs -o vers=4 kojihub:/external-repos /mnt/tmp/external-repos

Follow http://fedoraproject.org/wiki/Koji/ServerHowTo#Koji_Daemon_-_Builder

    mkdir /etc/pki/koji

Upload the following files from the hub:

    /etc/pki/koji/koji_ca_cert.crt
    /etc/pki/koji/kojibuilder2.pem
    chmod 644 /etc/pki/koji/*

Move data and prepare symlinks (root partition is too small)

    mkdir -p /mnt/tmp/var/lib
    mkdir -p /mnt/tmp/var/cache
    mv /var/lib/mock /mnt/tmp/var/lib/mock
    mv /var/tmp /mnt/tmp/var/tmp
    mv /var/cache/yum /mnt/tmp/var/cache/yum
    ln -s /mnt/tmp/var/lib/mock /var/lib/mock
    ln -s /mnt/tmp/var/tmp /var/tmp
    ln -s /mnt/tmp/var/cache/yum /var/cache/yum

Now edit /etc/kojid/kojid.conf

