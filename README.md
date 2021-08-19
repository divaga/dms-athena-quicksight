# DMS - Athena/Redshift - QuickSight Workshop

Migrate EC2 based postgresql to athena and visualize it thru QuickSight. Use for initial load only, not configured for CDC.

### Create VPC, IAM Role and Security Group

Create Elastic IP 

Create VPC with 2 private subnet

Create security group for ec2 (allow access from VPC subnet with standard postgresql port: 5432) and security group for Redshift

Create DMS IAM role 'dms-access-for-endpoint' with 'AmazonDMSRedshiftS3Role' policy

Create Glue role - AWSGlueServiceRole and S3 Access Policy

### If you want to use Redshift follow this

Create redshift subnet group 

Create redshift cluster (dc2), attach default service role, use our VPC (Welcome#1)

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

### Configure DMS

Create dms subnet group 

Create dms replication instance - note down internal ip addresss


### Allow access for DMS

Edit this file:
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

### Create DMS Endpoint

Create endpoint for source and target. For Athena as target use following additional setting:
```
addColumnName = true
DatePartitionEnabled = true
DatePartitionSequence = YYYYMMDD
DatePartitionDelimiter = DASH
```

Create replication task with initial load only and only for "public" schema

### Create Glue Crawler

Create and run glue crawler with correct S3 level

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

Add Athena as data source
