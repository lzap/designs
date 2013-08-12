Installation of Koji Daemon Builder
===================================

I am using Amazon EC2 instance (large). Create new instance in the same
datacenter as Koji Hub in the same security group (or with access to the 
hub I mean). I used RHEL 6.4 64bit system and attached one ephemeral
volume (it must be manually mounted - as /mnt/tmp).

    echo -e "p\nn\np\n1\n\n\np\nw\n" | fdisk /dev/xvdf
    mkfs.ext4 /dev/xvdf1
    mkdir /mnt/tmp
    echo "/dev/xvdf1 /mnt/tmp ext4 defaults 0 0" >> /etc/fstab
    mount -a

Note: The RHEL 6.4 AMI had an incorrect entry in /etc/fstab for some reason,
delete it:

    mount: special device /dev/xvdb does not exist

Copy our public ssh keys into /root/.ssh/authorizued_keys deleting the entry
there which disables root account.

Also add this entry to the /etc/hosts (must be the valid FQDN of the hub)

    echo "10.42.135.135 koji.katello.org" >> /etc/hosts

Prerequisites
-------------

We assume that two directories are exported on the hub (see /etc/exports on
the hub)

And we assume NFS ports are set and iptables rules too, see
http://lukas.zapletalovi.com/2011/05/export-for-both-nfs-v4-and-v3-clients.html
how to do that correctly.

Make sure the builder is in the same security group as the hub (which is
"default"), or configure security groups in a way that koji builder can access
NFS, 80 and 443 ports. To check NFS is working just do this (on the builder)

    showmount -e koji.katello.org
    Export list for koji.katello.org:
    /exports/external-repos *
    /exports/koji           *
    /exports                *

Before starting with configuration, we need to mount several directories from
the hub (see above)

    mkdir /mnt/koji
    mount -t nfs -o vers=4 koji.katello.org:/koji /mnt/koji
    mount -t nfs -o vers=4 koji.katello.org:/repos /mnt/koji/repos
    mkdir /mnt/tmp/external-repos
    mount -t nfs -o vers=4 koji.katello.org:/external-repos /mnt/tmp/external-repos

    echo "koji.katello.org:/koji /mnt/koji nfs defaults" >> /etc/hosts
    echo "koji.katello.org:/repos /mnt/koji/repos nfs defaults" >> /etc/hosts
    echo "koji.katello.org:/external-repos /mnt/tmp/external-repos nfs defaults" >> /etc/hosts

Installation
------------

Enable EPEL

    yum -y install http://mirror.slu.cz/epel/6/i386/epel-release-6-8.noarch.rpm

And install koji-builder package

    yum -y install koji-builder

We are done.

Configuration
-------------

Follow http://fedoraproject.org/wiki/Koji/ServerHowTo#Koji_Daemon_-_Builder
(instructions now follow):

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

    cat /etc/kojid/kojid.conf
    
    [kojid]
    mockhost=redhat-linux-gnu
    server = http://koji.katello.org/kojihub
    pkgurl = http://koji.katello.org/packages
    smtphost=localhost
    from_addr=Katello Koji Build System <buildsys@katello.org>
    cert = /etc/pki/koji/kojibuilder2.pem
    ca = /etc/pki/koji/koji_ca_cert.crt
    serverca = /etc/pki/koji/koji_ca_cert.crt

Due to older version of Koji on the hub, we need to downgrade koji-builder to
version 1.6.0-1. FIXME this should be published on fedorahosted.

    # service kojid stop
    # mkdir -p /var/opt/repos/RPMS/
    download koji-1.6.0 packages into /var/opt/repos/RPMS/
    # cd /var/opt/repos/RPMS/
    # createrepo .
    edit /etc/yum.repos.d/local.repo
    # yum --disablerepo=epel install koji-builder
    # mv /etc/kojid/kojid.conf.rpmsave /etc/kojid/kojid.conf
    # service kojid start

TODO: We can move the above repo to public and use that to install the
particular version.

Now, install this (hotfix) release which is the latest and greatest
createrepo written in C (this is not ever released in EPEL yet).

    download createrepo_c-0.1.17-3 /var/opt/repos/RPMS/
    # cd /var/opt/repos/RPMS/
    # createrepo .
    # yum install createrepo_c

TODO: The above can go away once
https://admin.fedoraproject.org/updates/FEDORA-EPEL-2013-11107/createrepo_c-0.2.0-1.el6
is published.

And change /usr/sbin/kojid to call createrepo_c instead of the Python version.

Add the new instance to the hub (do this as root on the hub shell)

    koji add-host kojibuilderN i386 x86_64
    koji add-host-to-channel kojibuilderN createrepo

We are done, each builder instance has capability to build and create build
roots via NFS.

