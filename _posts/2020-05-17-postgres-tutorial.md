---
layout: post
title: Getting started with PostgreSQL. How to create database and connect to it using psql shell.
slug: postgres-tutorial
keywords: postgresql postgres psql
description: How to create Postgres database and connect to it using psql shell.
---

PostgreSQL, also known as Postgres, is a free and open-source relational database
management system. psql is a terminal-based front-end to PostgreSQL. It enables 
to type in queries interactively, issue them to PostgreSQL, and see the query results. 

This guide is to give a basic understanding of how to work with Postgres. It explains
how to create a new database (local and Azure) and connect to it using psql shell. 

## Install

Follow this [official guide][1] to install Postgres on Ubuntu.

First, create the file repository configuration:

```shell
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

Then, import the repository signing key:

```shell
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

Update the package lists:

```bash
$ sudo apt-get update
```

And install the latest version of PostgreSQL:

```bash
$ sudo apt-get -y install postgresql
```

## Uninstall

The simplest way to do this is to open a terminal and type:

```bash
$ sudo apt-get --purge remove postgresql
```

This will also prompt you to remove that software that depends on Postgres.
It is possible that Postgres installs itself in several parts. In that case, a simple:

```bash
$ dpkg -l | grep postgres
```

Will get you the list of those packages that Postgres installed. Then, just use
the same ``apt-get --purge remove ....`` command but instead of just postgresql,
type each package name, separated by spaces, like:

```bash
$ sudo apt-get --purge remove postgresql-client-13 postgresql-client-common pgdg-keyring
```

As a next step, remove the following folders:

```bash
$ sudo rm -rf /var/lib/postgresql/
$ sudo rm -rf /var/log/postgresql/
$ sudo rm -rf /etc/postgresql/
```

And finally, remove the postgres user and group:

```bash
$ userdel -r postgres
$ groupdel postgres
```

That's it.

## Create a user

Once the installation is complete, you should add and configure a role for your
Ubuntu user so that you can easily enter the PostgreSQL environment and begin commanding the db.

First, sign into the default ``postgres`` user:

```bash
$ sudo su - postgres
postgres@alex:~$
```

Enter the PostgreSQL environment as follows:

```bash
postgres@alex:~$ psql
psql (13.0 (Ubuntu 13.0-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

Once you login you can view the Databases & Roles by giving the commands ``\list`` (list out the databases)
and ``\du`` (list out the roles).

```bash
postgres=# \list
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 |
 template0 | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

postgres=# \q
```

You can create a new role by using the below command from your terminal:

```bash
postgres@alex:~$ createuser --interactive
Enter name of role to add: alex
Shall the new role be a superuser? (y/n) y
postgres@alex:~$ psql
psql (13.0 (Ubuntu 13.0-1.pgdg20.04+1))
Type "help" for help.

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 alex      | Superuser, Create role, Create DB                          | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Use ``createuser --interactive --pwprompt`` to create a new role with password authentication.

You can remove a role using ``DROP ROLE`` statement:

```bash
postgres=# DROP ROLE alex;
```

## Create a database

Make sure you’re switched as a postgres user, if not use the below command ``sudo -i -u postgres``.

```bash
postgres@alex:~$ createdb research
```

When it's created, you can connect to it from shell:

```bash
$ psql -U alex -d research
research=#
```

To grant the connect access to the database, use following:

```bash
postgres=# GRANT CONNECT ON DATABASE dbname TO username;
```

## Create an Azure Database

Follow [this guide][2] to create an Azure Database for PostgreSQL server by using the Azure portal. 
When its created and the firewall rule configured, you can connect to the server via psql client.

> Use the empty database ``postgres`` with admin user.

Run the following command in shell terminal replacing values with your actual server name and admin user login name:

```bash
psql --host=mydemoserver.postgres.database.azure.com --port=5432 --username=myadmin@mydemoserver --dbname=postgres
```

List the available databases by issuing ``\list`` command:

```bash
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1), server 11.6)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=> \list
                                                               List of databases
       Name        |      Owner      | Encoding |          Collate           |           Ctype            |          Access privileges
-------------------+-----------------+----------+----------------------------+----------------------------+-------------------------------------
 azure_maintenance | azure_superuser | UTF8     | English_United States.1252 | English_United States.1252 | azure_superuser=CTc/azure_superuser
 azure_sys         | azure_superuser | UTF8     | English_United States.1252 | English_United States.1252 |
 postgres          | azure_superuser | UTF8     | English_United States.1252 | English_United States.1252 |
 template0         | azure_superuser | UTF8     | English_United States.1252 | English_United States.1252 | =c/azure_superuser                 +
                   |                 |          |                            |                            | azure_superuser=CTc/azure_superuser
 template1         | azure_superuser | UTF8     | English_United States.1252 | English_United States.1252 | =c/azure_superuser                 +
                   |                 |          |                            |                            | azure_superuser=CTc/azure_superuser
(5 rows)
```

Now you can create a new database.

```bash
postgres=> CREATE DATABASE research;
CREATE DATABASE
```

Mission completed!

## Create a new schema

To create a new schema, execute the following from the postgres shell:

```bash
postgres=# CREATE SCHEMA new_schema;
```

To give access to a user, use following:

```bash
postgres=# GRANT USAGE ON SCHEMA schema_name TO username;
postgres=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA schema_name TO username;
```

[1]: https://www.postgresql.org/download/linux/ubuntu/ "Install PostgreSQL on Ubuntu"
[2]: https://docs.microsoft.com/en-us/azure/postgresql/quickstart-create-server-database-portal "Create an Azure Database for PostgreSQL server by using the Azure portal"
