# DMS - Athena - QuickSight Workshop

Migrate EC2 based postgresql and mysql to athena and visualize it in near-realtime thru QuickSight. With initial load and CDC.

### Create VPC, IAM Role and Security Group

Create Elastic IP 

Create VPC with 2 private subnet

Create security group for ec2 (allow access from VPC subnet with standard postgresql port: 5432) 

Create DMS IAM role 'dms-access-for-endpoint' with 'AmazonDMSRedshiftS3Role' policy

Create Glue role - AWSGlueServiceRole and S3 Access Policy

### If you want to use Athena follow this
Create bucket with "dms-" prefix


### Install PostgreSQL as data source

Create EC2, use t3.small and use our VPC


Install postgreSQL
```
sudo yum update
sudo amazon-linux-extras enable postgresql11
sudo yum install postgresql postgresql-devel postgresql-server postgresql-contrib
sudo postgresql-setup initdb
sudo systemctl enable --now postgresql 
systemctl status postgresql
sudo yum install git
```

Configure user
```
sudo su - postgres
psql -c "alter user postgres with password 'yourstrongpassword'"
psql -c "create database posindo"
psql -c "ALTER USER postgres WITH SUPERUSER"
```

Install sample data
```
git clone https://github.com/devrimgunduz/pagila.git
cd pagila 
psql posindo -f pagila-schema.sql
psql posindo -f pagila-data.sql
```

## Install MySQL as second data source

Create EC2, use t3.small and same VPC

Install MySQL

```
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
sudo amazon-linux-extras install epel -y
sudo yum install mysql-community-server
sudo systemctl enable --now mysqld
systemctl status mysqld
sudo grep 'temporary password' /var/log/mysqld.log
```

You will get temporary password, please note down and execute this:

```
sudo mysql_secure_installation -p'<your_temp_password>'
```

Follow the steps and note to disallow root login remotely = No 


Initiate sample database

```
wget https://downloads.mysql.com/docs/sakila-db.zip
```
unzip this sakila-db.zip and change directory to sakila-db


Prepare user
```
mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Welcome#1';
CREATE USER 'dms'@'%' IDENTIFIED WITH mysql_native_password BY 'Welcome#1';
GRANT ALL PRIVILEGES ON *.* TO 'dms'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Change user

```
mysql -udms -p
source sakila-schema.sql;
source sakila-data.sql;
```

Modify MySQL Server parameter
```
sudo vi /etc/my.cnf
```

Add these following lines
```
server-id= 1
log-bin=bin.log
binlog_format=ROW
expire_logs_days=1
binlog_checksum=NONE
binlog_row_image=FULL
log_slave_updates=FALSE
bind-address=0.0.0.0
```


### Configure DMS

Create dms subnet group 

Create dms replication instance - note down internal ip addresss


### Allow access for DMS

Edit this file in PostgreSQL host:
```
sudo su - postgres 
vi /var/lib/pgsql/data/pg_hba.conf
```
and allow your DMS IP address
```
# Replication Instance
host all all 12.3.4.56/00 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
host replication postgres 12.3.4.56/00 md5
```
and also this file:
```
vi /var/lib/pgsql/data/postgresql.conf
```

change:
```
listen_addresses = '*'

wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
wal_sender_timeout = 0


shared_preload_libraries= 'test_decoding'
```

Restart
```
sudo systemctl restart postgresql
```

### Configure Replication for PostgreSQL DB

Execute this following SQL Statement on source PG database:

```
SELECT * FROM pg_create_logical_replication_slot('my_replication_slot', 'test_decoding'); 
```

validate by executing this:
```
select * from pg_replication_slots
```


### Create DMS Endpoint

Create endpoint for database source and S3 target endpoint (for both MySQL and PostgreSQL). For S3 as target use following additional setting:

```
addColumnName = true
DatePartitionEnabled = true
DatePartitionSequence = YYYYMMDD
DatePartitionDelimiter = DASH
dataFormat = parquet
```

Create 2 replication task (for PostgreSQL and MySQL as data source) with initial load and replicate on going changes, and only for "public" schema (PostgreSQL) and "sakila" schema (MySQL)

### Create Glue Crawler

Create and run glue crawler with correct S3 folder level

### Configure Athena
Configure Athena S3 setting

Create this view

```
create view actor_summary as 
select actor.first_name, actor.last_name,
count(actor.actor_id) as film_count
from actor actor ,film_actor film_actor
where actor.actor_id = film_actor.actor_id
group by actor.first_name, actor.last_name
```

### Configure QuickSight
Sign up to QuickSight 

Gave S3 access 

Enable QuickSight access to Athena and S3

Add Athena as data source and disable SPICE

Create visualization
