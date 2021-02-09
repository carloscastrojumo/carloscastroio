---
title: "AWS RDS migration with Bucardo"
date: 2021-02-09T13:48:43Z
author: "Carlos Castro"
tags: ["AWS RDS", "PostgreSQL", "Bucardo"]
categories: ["AWS"]
description: "How to migrate AWS RDS databases with Bucardo"
draft: true
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true 
disableShare: true
disableHLJS: true
searchHidden: true
cover:
    image: "https://miro.medium.com/max/367/1*6j17ZDuywkKu7TOa2yvAKg.png"
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

After struggling with migrate data from one AWS RDS database into another with Bucardo, with multiple different errors during the way, I decided to create my own guide.

This tutorial was tested with an AWS EC2 with Ubuntu 18.04 and two AWS RDS Aurora PostgreSQL 10.11 (one with data, other with a clean installation).

## What is Bucardo
[Bucardo](https://bucardo.org/) is an asynchronous PostgreSQL replication system, allowing for both multi-master and multi-slave operations.

We will use Bucardo for migrate data from one AWS RDS to another without downtime. 

## Install and configure PostgreSQL for Bucardo

Bucardo needs it's own database to save configuration and migration state. Let's install it on the same machine where Bucardo will be running.

```sh
# Add PostgreSQL repository configuration:
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update the package lists:
$ sudo apt-get update

# install PostgreSQL 13
$ sudo apt install postgresql-13 postgresql-plperl-13 -y

# start PostgreSQL 13
$ sudo pg_ctlcluster 13 main start

```

Edit file */etc/postgresql/13/main/pg_hba.conf* and add the following line:
```sh
# Database administrative login by Unix domain socket
local   all             bucardo                                 trust
```

Restart PostgreSQL service.
```sh
$ sudo systemctl restart postgresql
```

Alternatively, create a `pgpass` file
```sh
echo "127.0.0.1:5432:bucardo:bucardo:strongPassword" > $HOME/.pgpass  
chmod 0600 $HOME/.pgpass  
```

Create user bucardo as superuser in local PostgreSQL.
```sh
sudo su - postgres -c "psql postgres"
CREATE USER bucardo WITH LOGIN SUPERUSER ENCRYPTED PASSWORD 'strongPassword';
CREATE DATABASE bucardo OWNER=bucardo;
\q
```

## Install Bucardo

**Note:** I decided to go with the source code version and compile it instead of Ubuntu package version because the last one comes with PostgreSQL 9.4 as a dependency.

Start by installing Bucardo dependencies.
```sh 
$ sudo apt install make libdbix-safe-perl libdbd-pg-perl libencode-locale-perl libboolean-perl -y
```

Required directories and ownerships.
```sh
sudo mkdir /var/run/bucardo
sudo mkdir /var/log/bucardo
sudo chown ubuntu:ubuntu /var/run/bucardo
sudo chown ubuntu:ubuntu /var/log/bucardo
```

Download and install Bucardo.
```sh
$ wget https://bucardo.org/downloads/Bucardo-5.6.0.tar.gz
$ tar -zxf Bucardo-5.6.0.tar.gz
$ cd Bucardo-5.6.0
$ sudo perl Makefile.PL
$ sudo make
```

Finally, execute `bucardo install` command and confirm if the parameters are correct. 

```sh
Current connection settings:
1. Host:           localhost
2. Port:           5432
3. User:           bucardo
4. Database:       bucardo
5. PID directory:  /var/run/bucardo
Enter a number to change it, P to proceed, or Q to quit: P
```

If everything goes ok, you should see a successful message.
```sh
Attempting to create and populate the bucardo database and schema
Database creation is complete

Updated configuration setting "piddir"
Installation is now complete.
If you see errors or need help, please email bucardo-general@bucardo.org

You may want to check over the configuration variables next, by running:
bucardo show all
Change any setting by using: bucardo set foo=bar
```

Bucardo is ready to be used.

## Dump from current RDS to new one

Before start syncing data we need to have both databases with the same schemas.

For simplify, let's declare the following variables for the source database:

```sh
$ export SRC_HOST=db-source.ctybapsaji9w.eu-west-1.rds.amazonaws.com
$ export SRC_PORT=5432
$ export SRC_DB=my_db
$ export SRC_USER=my_user
$ export SRC_PASS=my_pass 
```
and for destination database.

```sh
$ export DST_HOST=db-destination.ctybapsaji9w.eu-west-1.rds.amazonaws.com
$ export DST_PORT=5432
$ export DST_DB=my_db
$ export DST_USER=my_user
$ export DST_PASS=my_pass
```

Take a `dump` of the tables schema you want want to migrate (eg: phonebook).
```sh
$ pg_dump "host=$SRC_HOST port=$SRC_PORT dbname=$SRC_DB user=$SRC_USER" -t "public.phonebook" --schema-only | grep -v 'CREATE TRIGGER' | grep -v '^--' | grep -v '^$' | grep -v '^SET' | grep -v 'OWNER TO' > schema.sql  
```

Create the database in the destination AWS RDS.
```sh
$ psql "host=$DST_HOST port=$DST_PORT dbname=postgres user=$DST_USER" -c "CREATE DATABASE $DST_DB;"
```

Next, import the dumped schemas.
```sh
$ psql "host=$DST_HOST port=$DST_PORT dbname=$DST_DB user=$DST_USER" -f schema.sql
```

## Configure Bucardo for replication

With database and schemas created in the destination database, the final step is to copy the tables data from one database to another.

Add the source and destionation databases into Bucardo.
```sh
$ bucardo add db src_db dbhost=$SRC_HOST dbport=$SRC_PORT dbname=$SRC_DB dbuser=$SRC_USER dbpass=${SRC_PASS}
$ bucardo add db dst_db dbhost=$DST_HOST dbport=$DST_PORT dbname=$DST_DB dbuser=$DST_USER dbpass=${DST_PASS}
```

Add the tables to be migrated from the source database (list separated by space).
```sh
$ bucardo add tables public.phonebook db=src_db
```

Create the herd. Herds are used to designate the “source” of a sync.

```sh
$ bucardo add herd copying_herd public.phonebook
```

Create the sync.

```sh
$ bucardo add sync the_sync relgroup=copying_herd dbs=src_db:source,dst_db:target onetimecopy=2
```

Start Bucardo.
```sh
$ bucardo start
```

To see the status we can do:
```sh
$ bucardo status
```

Or tail the log `tail -f /var/log/bucardo/log.bucardo`

You should start seeing data replicated from source database in destionation database.
