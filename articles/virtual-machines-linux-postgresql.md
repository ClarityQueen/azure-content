<properties
	pageTitle="Install and configure PostgreSQL on a Microsoft Azure virtual machine running Linux"
	description="Learn how to install and configure PostgreSQL on an Ubuntu or CentOS virtual machine (VM) in Azure."
	services="virtual-machines"
	documentationCenter=""
	authors="SuperScottz"
	manager="timlt"
	editor=""
  tags=""/>

<tags
	ms.service="virtual-machines"
	ms.devlang="na"
	ms.topic="article"
	ms.tgt_pltfrm="linux"
	ms.workload="infrastructure-services"
	ms.date="06/04/2015"
	ms.author="mingzhan"/>


#Install and configure PostgreSQL on Microsoft Azure

PostgreSQL is an advanced open-source database similar to Oracle and DB2. It includes enterprise-ready features such as full ACID compliance, reliable transactional processing, and multiversion concurrency control. It also supports standards such as ANSI SQL and SQL/MED (including foreign data wrappers for Oracle, MySQL, MongoDB, and many others). It is highly extensible with support for over 12 procedural languages, GIN and GIST indexes, spatial data support, and multiple NoSQL-like features for JSON or key-value based applications.

In this tutorial, you will learn how to install and configure PostgreSQL on a Microsoft Azure virtual machine running Linux.

> [Azure.NOTE] You must already have an Azure virtual machine running Linux to complete this tutorial. If you don't, please see 
[Create a Virtual Machine Running Linux](virtual-machines-linux-tutorial.md) before proceeding.

[In this case, use port 1999 as the PostgreSQL port.]  

## Install PostgreSQL

Use PuTTY to connect to the Azure virtual machine running Linux that you created. If this is the first time you are using this virtual machine, you can learn how to use PuTTY to connect in the following article: [How to Use SSH with Linux on Azure](virtual-machines-linux-use-ssh-key.md).

1. Run the following command to switch to the root directory (admin):

		$ sudo su -

2. Some distributions have dependencies that you must install before you install PostgreSQL. Run the appropriate command from the following list:

	- Redhat:

			# yum install readline-devel gcc make zlib-devel openssl openssl-devel libxml2-devel pam-devel pam  libxslt-devel tcl-devel python-devel -y  

	- Debian:

 			# apt-get install readline-devel gcc make zlib-devel openssl openssl-devel libxml2-devel pam-devel pam libxslt-devel tcl-devel python-devel -y  

	- Suse:

			# zypper install readline-devel gcc make zlib-devel openssl openssl-devel libxml2-devel pam-devel pam  libxslt-devel tcl-devel python-devel -y  

3. Download PostgreSQL to the root directory, and then unzip the package:

		# wget https://ftp.postgresql.org/pub/source/v9.3.5/postgresql-9.3.5.tar.bz2 -P /root/

		# tar jxvf  postgresql-9.3.5.tar.bz2

	The previous code provides an example. To find a detailed download address, see [Create a Virtual Machine Running Linux](https://ftp.postgresql.org/pub/source/).

4. To start the build, run these commands:

		# cd postgresql-9.3.5

		# ./configure --prefix=/opt/postgresql-9.3.5

5. If  you want to build everything, including the documentation (HTML and main pages) and additional modules (contrib), run the following command instead:

		# gmake install-world

	You should receive the following confirmation message:

		PostgreSQL, contrib, and documentation successfully made. Ready to install.

## Configure PostgreSQL

1. (Optional) Create a symbolic link to shorten the PostgreSQL reference so it does not include the version number:

		# ln -s /opt/pgsql9.3.5 /opt/pgsql

2. Create a directory for the database:

		# mkdir -p /opt/pgsql_data

3. Create a user profile that is not in the root directory and modify that profile. Then switch to this user (called *postgres* in our example):

		# useradd postgres

		# chown -R postgres.postgres /opt/pgsql_data

		# su - postgres

    >[Azure.NOTE] For security reasons, PostgreSQL uses a user profile that is not associated with the root directory to initialize, start, or shut down the database.


4. Edit the *bash_profile* by adding the following commands to the end of the *bash_profile* file:

		cat >> ~/.bash_profile <<EOF
		export PGPORT=1999
		export PGDATA=/opt/pgsql_data
		export LANG=en_US.utf8
		export PGHOME=/opt/pgsql
		export PATH=\$PATH:\$PGHOME/bin
		export MANPATH=\$MANPATH:\$PGHOME/share/man
		export DATA=`date +"%Y%m%d%H%M"`
		export PGUSER=postgres
		alias rm='rm -i'
		alias ll='ls -lh'
		EOF

5. Run the *bash_profile* file:

		$ source .bash_profile

6. Validate your installation with the following command:

		$ which psql

	If your installation is successful, you will see the following response:

		/opt/pgsql/bin/psql

7. You can also check the PostgreSQL version:

		$ psql -V

8. Initialize the database:

		$ initdb -D $PGDATA -E UTF8 --locale=C -U postgres -W

	You should receive the following output:

![image](./media/virtual-machines-linux-postgresql/no1.png)

## Set up PostgreSQL

<!--	[postgres@ test ~]$ exit -->

Run the following commands:

	# cd /root/postgresql-9.3.5/contrib/start-scripts

	# cp linux /etc/init.d/postgresql

Modify two variables in the /etc/init.d/postgresql file. The prefix is set to the installation path of PostgreSQL: **/opt/pgsql**. PGDATA is set to the data storage path of PostgreSQL: **/opt/pgsql_data**.

	# sed -i '32s#usr/local#opt#' /etc/init.d/postgresql

	# sed -i '35s#usr/local/pgsql/data#opt/pgsql_data#' /etc/init.d/postgresql

![image](./media/virtual-machines-linux-postgresql/no2.png)

Change the file to make it executable:

	# chmod +x /etc/init.d/postgresql

Start PostgreSQL:

	# /etc/init.d/postgresql start

Check if the endpoint of PostgreSQL is on:

	# netstat -tunlp|grep 1999

You should see the following output:

![image](./media/virtual-machines-linux-postgresql/no3.png)

## Connect to the PostgresSQL database

Switch to the PostgresSQL user again:

	# su - postgres

Create a PostgresSQL events database:

	$ createdb events

Connect to the events database that you just created:

	$ psql -d events

## Create and delete a PostgresSQL table

Now that you have connected to the database, you can create tables in it.

For example, create a PostgresSQL table with the following command:

	CREATE TABLE potluck (name VARCHAR(20),	food VARCHAR(30),	confirmed CHAR(1), signup_date DATE);

This sets up a four-column table with the following column names and restrictions:

1. The “name” column is limited by the VARCHAR command to be under 20 characters long.
2. The “food” column indicates the food item that each person will bring. VARCHAR limits this text to under 30 characters.
3. The “confirmed” column records whether the person has responded to the potluck invitation. The acceptable values are "Y" and "N".
4. The “date” column shows when they signed up for the event. PostgresSQL requires that dates be written as yyyy-mm-dd.

You should see the following if your table is successfully created:

![image](./media/virtual-machines-linux-postgresql/no4.png)

You can also check the table structure with the following command:

![image](./media/virtual-machines-linux-postgresql/no5.png)

### Add data to a table

Use the following command to insert information into a row:

	INSERT INTO potluck (name, food, confirmed, signup_date) VALUES('John', 'Casserole', 'Y', '2012-04-11');

You should see this output:

![image](./media/virtual-machines-linux-postgresql/no6.png)

You can also add more information to the table. Here are some options, or you can create your own:

	INSERT INTO potluck (name, food, confirmed, signup_date) VALUES('Sandy', 'Key Lime Tarts', 'N', '2012-04-14');

	INSERT INTO potluck (name, food, confirmed, signup_date) VALUES ('Tom', 'BBQ','Y', '2012-04-18');

	INSERT INTO potluck (name, food, confirmed, signup_date) VALUES('Tina', 'Salad', 'Y', '2012-04-18');

### Show tables

Use the following command to show a table:

	select * from potluck;

The output is:

![image](./media/virtual-machines-linux-postgresql/no7.png)

### Delete data in a table

Use the following command to delete data in a table:

	delete from potluck where name=’John’;

This will delete all the information in the "John" row. The output is:

![image](./media/virtual-machines-linux-postgresql/no8.png)

### Update data in a table

Use the following command to update data in a table. In this example, Sandy has confirmed that she is attending, so the command changes her RSVP from "No" to "Yes":

 	UPDATE potluck set confirmed = 'Y' WHERE name = 'Sandy';


##More regarding PostgreSQL
In this tutorial, you learned how to install and configure PostgreSQL in a Microsoft Azure virtual machine running Linux. Enjoy your journey using it in Azure. For more information about PostgreSQL, see the [PostgreSQL website](http://www.postgresql.org/).
