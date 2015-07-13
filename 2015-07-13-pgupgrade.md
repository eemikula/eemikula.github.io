---
layout: post
title: Upgrading Postgres from Fedora 21 to 22
---

I recently upgraded my personal computer from Fedora 21 to 22, and in the process ran into some hiccups with my Postgres server. Since it was such a nuisance to bring it back up correctly after the upgrade to Postgres 9.4, I wanted to document the steps here: I'm sure I'll need these notes later when I have to perform a similar upgrade on a different machine.

The problem stems from a variety of different things: Firstly, the Fedora upgrade will not automatically install the necessary components to upgrade an existing database: It will only upgrade the server itself, which will then fail to start against an old database format. Additionally, my situation was further complicated because of the use of PostGIS, which will still be absent even after installing the postgresql-upgrade package. Finally, certain configuration settings in pg_hba.conf and postgresql.conf can cause the upgrade to fail, particularly around SSL settings.

The following is my list of steps, derived from [this blog post](http://tso.bzb.us/2015/05/postgresql-upgrade-fedora-22.html), with a few changes necessitated by my own configuration (especially around postgis). Note that it depends on a copy of postgis-2.1.so, which is extracted from a Fedora 21 RPM (e.g. the [RPM found here](http://koji.fedoraproject.org/koji/buildinfo?buildID=625650)). This is copied into the postgis-9.3 directory so that it can be used by the upgrade.

```

dnf install postgresql-upgrade
mv /var/lib/pgsql/data /var/lib/pgsql/data_9.3

# to be extracted from an RPM from the previous version of the OS
cp /root/postgis-2.1.so `pg_config --pkglibdir`/postgresql-9.3/lib

# md5 logins and certificates can cause upgrade to fail, so disable these for the duration of the upgrade.
# They will need to be manually reverted when the upgrade is completed.
sed -i "s/md5$/trust/" /var/lib/pgsql/data_9.3/pg_hba.conf
sed -i "s/.*cert$/#&/" /var/lib/pgsql/data_9.3/pg_hba.conf
sed -i "s/^ssl = on/#ssl = on/" /var/lib/pgsql/data_9.3/postgresql.conf

postgresql-setup initdb
cp /var/lib/pgsql/data_9.3/pg_hba.conf /var/lib/pgsql/data/pg_hba.conf

su - postgres
cd /var/lib/pgsql
pg_upgrade -b /usr/lib64/pgsql/postgresql-9.3/bin/ -B /usr/bin/ -d data_9.3/ -D data

systemctl start postgresql
./analyze_new_cluster.sh
./delete_old_cluster.sh

```

