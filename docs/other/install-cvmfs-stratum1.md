title: Install a CVMFS Stratum 1

# Install a CVMFS Stratum 1

This document describes how to install a CVMFS Stratum 1. There are many different variations on how to do that, but this document focuses on the configuration of the OSG Operations Stratum 1 oasis-replica.opensciencegrid.org. It is applicable to other Stratum 1s as well, very likely with modifications (some of which are suggested in the document below).

!!! note "Applicable versions"
    The applicable software versions for this document are cvmfs and cvmfs-server >= 2.4.2.

## Before Starting

Before starting the installation process, consider the following points:

- **User IDs and Group IDs:** If your machine is also going to be a repository server like OSG Operations, the installation will create the same user and group IDs as the [cvmfs client](../worker-node/install-cvmfs.md).  If you are installing frontier-squid, the installation will also create the same user id as [frontier-squid](../data/frontier-squid.md).
-  **Network ports:** This installation will host the stratum 1 on ports 80, 8000 and 8080, and if squid is installed it will host the uncached apache on port 8081.  Port 80 is default but sometimes runs into operational problems, port 8000 is the alternate for most production use, and port 8080 is for Cloudflare (https://openhtc.io).
- **Host choice:** -  Make sure there is adequate disk space for all the repositories that will be served, at `/srv/cvmfs`. In addition, about 100GB should be reserved for apache and squid logs under /var/log on a production server, although they normally will not get that large.  Apache logs get larger than squid logs because by default they are rotated much less frequently.  Many installations share that space with the filesystem used for /srv/cvmfs by turning that directory along with /var/log/squid and /var/log/httpd into symlinks pointing to directories on the big filesystem.

As with all OSG software installations, there are some one-time (per host) steps to prepare in advance:

- Ensure the host has [a supported operating system](../release/supported_platforms.md)
- Obtain root access to the host
- Prepare the [required Yum repositories](../common/yum.md)

## Installing

All CVMFS Stratum 1s require cvmfs-server software and apache (httpd). It is highly recommended to also install [frontier-squid](../data/frontier-squid.md) and [frontier-awstats](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallAwstats) on the same machine to be able to easily join the WLCG [MRTG](http://wlcg-squid-monitor.cern.ch/snmpstats/indexcvmfs.html) and [awstats](http://wlcg-squid-monitor.cern.ch/awstats/cvmfs.html) monitoring systems. The recommended configuration for frontier-squid below only caches geo api lookups.  Other than that, it is primarily for monitoring.

### Installing cvmfs-server and httpd

Use this command to install cvmfs-server and httpd:

```console
root@host # yum -y install cvmfs-server cvmfs-config mod_wsgi
```

On EL8 use `python3-mod_wsgi` instead of `mod_wsgi`.

### Installing frontier-squid and frontier-awstats

[frontier-awstats](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallAwstats) is not distributed by OSG so these instructions get it from its original source.  Do these commands to install frontier-squid and frontier-awstats:

```console
root@host # rpm -i http://frontier.cern.ch/dist/rpms/RPMS/noarch/frontier-release-1.2-1.noarch.rpm
root@host # yum -y install frontier-squid frontier-awstats
```

## Configuring

### Configuring the system

Increase the default number of open file descriptors:

```console
root@host # echo -e "*\t\t-\tnofile\t\t16384" >>/etc/security/limits.conf 
root@host # ulimit -n 16384
```

In order for this to apply also interactively when logging in over ssh, the option `UsePAM` has to be set to `yes` in `/etc/ssh/sshd_config`.

### Configuring cron 
First, create the log directory: 

```console
root@host # mkdir -p /var/log/cvmfs
```

Put the following in `/etc/cron.d/cvmfs`:

```
*/5 * * * * root test -d /srv/cvmfs || exit;cvmfs_server snapshot -ai 
6 1 * * * root cvmfs_server gc -af 2>/dev/null || true
0 9 * * * root find /srv/cvmfs/*.*/data/txn -name "*.*" -mtime +2 2>/dev/null|xargs rm -f
```

Also, put the following in `/etc/logrotate.d/cvmfs`:

```
/var/log/cvmfs/*.log {
    weekly
    missingok
    notifempty
}
```

### Configuring apache

If you are installing frontier-squid, create `/etc/httpd/conf.d/cvmfs.conf` and put the following lines into it:

```
Listen 8081
KeepAlive On
```

If you are not installing frontier-squid, instead put the following lines into that file:

```
Listen 8000
Listen 8080
KeepAlive On
```

Then enable apache:

```console
root@host # systemctl enable httpd
root@host # systemctl start httpd
```


### Configuring frontier-squid

Put the following in `/etc/squid/customize.sh` after the existing comment header:

```awk
awk --file `dirname $0`/customhelps.awk --source '{

# cache only api calls 
insertline("^http_access deny all", "acl CVMFSAPI urlpath_regex ^/cvmfs/[^/]*/api/")
insertline("^http_access deny all", "cache deny !CVMFSAPI")

# port 80 is also supported, through an iptables redirect 
setoption("http_port", "8080 accel defaultsite=localhost:8081 no-vhost")
insertline("^http_port","http_port 8000 accel defaultsite=localhost:8081 no-vhost")
setoption("cache_peer", "localhost parent 8081 0 no-query originserver")

# allow incoming http accesses from anywhere
# all requests will be forwarded to the originserver 
commentout("http_access allow NET_LOCAL")
insertline("^http_access deny all", "http_access allow all")

# do not let squid cache DNS entries more than 5 minutes 
setoption("positive_dns_ttl", "5 minutes")

# set shutdown_lifetime to 0 to avoid giving new connections error
# codes, which get cached upstream 
setoption("shutdown_lifetime", "0 seconds")

# turn off collapsed_forwarding to prevent slow clients from slowing down
# faster ones
setoption("collapsed_forwarding", "off")

print
}'
```

Set up the firewall to allow incoming tcp ports 8000 & 8080, 
incoming udp port 3401, and forward port 80 to 8000.
There are various ways to do it, but these instructions assume that
firewalld is enabled and started:

```console
root@host # firewall-cmd --permanent --add-port=8000/tcp
root@host # firewall-cmd --permanent --add-port=8080/tcp
root@host # firewall-cmd --permanent --add-port=3401/udp
root@host # firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8000
root@host # firewall-cmd --reload
```

For nftables or iptables see these alternate
[instructions for having squid listen on a privileged
port](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Having_squid_listen_on_a_privile).

Enable frontier-squid:

```console
root@host # systemctl enable frontier-squid
root@host # systemctl start frontier-squid
```

!!! note
    The above configuration is for a single squid thread, which is fine for 1Gbit/s and possibly 2Gbit/s, but if higher bandwidth is needed, see the [instructions for running multiple squid workers](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Running_multiple_squid_workers).

## Verifying

In order to verify that everything is installed correctly, create a repository replica. The repository chosen for the instructions below is the OSG config repository because it is very small, but you can use another one if you prefer.

### Adding an example repository

It's a good idea to make your own script for adding repository replicas, because there's always at least two commands to run, and it's easy to forget which commands to run. The commands are:

```console
root@host # cvmfs_server add-replica -o root http://oasis.opensciencegrid.org:8000/cvmfs/config-osg.opensciencegrid.org /etc/cvmfs/keys/opensciencegrid.org/opensciencegrid.org.pub
root@host # cvmfs_server snapshot config-osg.opensciencegrid.org
```

With large repositories that can take a very long time, but with small repositories it should be very quick and not show any errors.

### Verifying that the replica is being served

Now to verify that the replication is working, do the following commands:

```console
root@host # wget -qdO- http://localhost:8000/cvmfs/config-osg.opensciencegrid.org/.cvmfspublished | cat -v
root@host # wget -qdO- http://localhost:80/cvmfs/config-osg.opensciencegrid.org/.cvmfspublished | cat -v
```

Both commands should show a short file including gibberish at the end which is the signature.

It is a good idea to familiarize yourself with the log entries at `/var/log/httpd/access_log` and also, if you have installed frontier-squid, at `/var/log/squid/access.log`. Also, at least 15 minutes after the snapshot is finished, check the log `/var/log/cvmfs/snapshots.log` to see that it tried to get an update and got no errors.

## Setting up monitoring

If you installed frontier-squid and frontier-awstats, there is a little more to do to configure monitoring.

First, make sure that your firewall accepts UDP queries from the monitoring server at CERN. Details are in [the frontier-squid instructions](https://twiki.cern.ch/twiki/bin/view/Frontier/InstallSquid#Enabling_monitoring). 

Next, choose any random password and put it in `/etc/awstats/password-file`. Then tell Dave Dykstra the fully qualified domain name of your machine and the password you chose, and he'll set up the monitoring servers.

Finally, install the [cvmfs-servermon package](https://github.com/cvmfs-contrib/cvmfs-servermon#readme) so the stratum 1 can be watched for problems with repositories.

## Managing replication

Instead of manually managing replication it is highly recommended to use the [cvmfs-manage-replicas package](https://github.com/cvmfs-contrib/cvmfs-manage-replicas#readme) which can automatically add repositories based on wildcards of repositories installed elsewhere.

