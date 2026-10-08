# AWS DMS: MySQL to Amazon RDS Migration Lab

## 📌 Project Overview

This lab demonstrates how to migrate a MySQL database running locally on a Windows machine to **Amazon RDS for MySQL** using **AWS Database Migration Service (AWS DMS)**.

The local MySQL environment was used to **simulate an on-premises source database**.

The migration was configured as:

> **Full Load + Change Data Capture (CDC)**

This means AWS DMS first migrated the existing database contents and then continued replicating changes made to the source database.

---

## 🎯 Lab Objectives

The objectives of this lab were to:

* Simulate an on-premises MySQL source environment.
* Configure MySQL for AWS DMS.
* Enable binary logging and CDC.
* Establish secure connectivity between the local database and AWS.
* Create an Amazon RDS for MySQL target database.
* Configure AWS DMS source and target endpoints.
* Perform a Full Load migration.
* Enable ongoing CDC replication.
* Verify that changes made to the source database are replicated.
* Document the troubleshooting process and final architecture.

---

## 🏗️ Architecture

```text
┌──────────────────────────────┐
│       Windows Laptop         │
│                              │
│   MySQL 8.0                  │
│   Database: shop             │
│                              │
│   customers                  │
│   products                   │
│   orders                     │
└──────────────┬───────────────┘
               │
               │ Tailscale
               │
               ▼
┌──────────────────────────────┐
│        EC2 Bridge            │
│                              │
│   Amazon Linux 2023          │
│   t3.micro                   │
│                              │
│   socat TCP Proxy            │
│   Port 3306                  │
└──────────────┬───────────────┘
               │
               │ Private VPC
               ▼
┌──────────────────────────────┐
│          AWS DMS             │
│                              │
│ shop-dms-replication         │
│ DMS t3.medium                │
│                              │
│ Full Load + CDC              │
└──────────────┬───────────────┘
               │
               │ Private VPC
               ▼
┌──────────────────────────────┐
│       Amazon RDS             │
│                              │
│   MySQL                      │
│   shop-dms-target            │
│   db.t3.micro                │
└──────────────────────────────┘
```

### Connectivity Design

The source MySQL database was not directly exposed to the public internet.

The laptop was behind a CGNAT connection, so inbound connectivity from AWS to the laptop was not practical.

Instead, Tailscale provided connectivity between the laptop and an EC2 bridge instance.

The EC2 instance used `socat` as a TCP proxy:

```text
DMS
 ↓
EC2 private IP:3306
 ↓
socat
 ↓
Tailscale IP:3306
 ↓
Local Windows MySQL
```

---

# 🗄️ Source Database

## MySQL Environment

* Platform: Windows
* MySQL: 8.0
* Database: `shop`

### Tables

The database contains three tables:

```text
customers
products
orders
```

### Customers

```text
customer_id
name
email
created_at
```

### Products

```text
product_id
name
price
stock
```

### Orders

```text
order_id
customer_id
product_id
quantity
order_date
```

Foreign keys were configured between `orders`, `customers`, and `products`.

---

# 🔄 CDC Configuration

AWS DMS CDC requires MySQL binary logging.

The following settings were verified:

```sql
SHOW GLOBAL VARIABLES LIKE 'log_bin';
SHOW GLOBAL VARIABLES LIKE 'binlog_format';
SHOW GLOBAL VARIABLES LIKE 'binlog_row_image';
```

The final values were:

```text
log_bin          ON
binlog_format    ROW
binlog_row_image FULL
```

MySQL network timeout values were also increased globally to support the migration:

```sql
SET GLOBAL net_read_timeout = 300;
SET GLOBAL net_write_timeout = 300;
```

---

# 👤 DMS Source User

A dedicated MySQL user was created for AWS DMS rather than using the local root account.

Example privileges:

```sql
CREATE USER 'dms_user'@'%' IDENTIFIED BY '<password>';

GRANT REPLICATION SLAVE,
      REPLICATION CLIENT,
      RELOAD
ON *.*
TO 'dms_user'@'%';

GRANT SELECT,
      SHOW VIEW,
      EVENT,
      TRIGGER
ON shop.*
TO 'dms_user'@'%';
```

The password is intentionally not stored in this repository.

---

# ☁️ AWS Resources

## Amazon RDS

Target database:

```text
Identifier: shop-dms-target
Engine: MySQL
Instance class: db.t3.micro
Region: eu-north-1
Multi-AZ: No
Public access: No
```

The RDS instance was kept private inside the VPC.

---

## AWS DMS

Replication instance:

```text
Identifier: shop-dms-replication
Class: dms.t3.medium
Region: eu-north-1
Multi-AZ: No
Publicly accessible: No
```

### Source Endpoint

```text
Identifier: shop-mysql-source
Engine: MySQL
Server: EC2 private IP
Port: 3306
SSL: None
```

The endpoint connects to the EC2 bridge, which forwards the traffic to the local MySQL instance through Tailscale.

### Target Endpoint

```text
Identifier: shop-mysql-target
Engine: MySQL
Server: RDS endpoint
Port: 3306
SSL: None
```

Both DMS endpoint connection tests completed successfully.

---

# 🚀 Migration Task

The migration task was configured as:

```text
Task: shop-mysql-migration

Migration type:
Full Load + CDC

Table mapping:
shop.*

Target table preparation:
Drop tables on target

LOB mode:
Do not include LOB columns

Data validation:
Enabled

CloudWatch logs:
Enabled
```

---

# ✅ Migration Result

The AWS DMS task successfully completed the initial Full Load.

Final task status:

```text
Load completed, replication ongoing
```

Migration results:

```text
Tables loaded:    3
Tables errored:   0
Tables queued:    0
```

The three source tables were successfully migrated:

```text
customers
products
orders
```

After the Full Load completed, the task remained in:

```text
replication ongoing
```

This confirmed that DMS had transitioned to the CDC phase.

---

# 🔥 CDC Verification

To verify CDC, a new record was inserted into the source MySQL database **after the Full Load had already completed**.

Example:

```sql
USE shop;

INSERT INTO customers (name, email)
VALUES ('CDC Test', 'cdc-test@example.com');
```

The new record was subsequently reflected in the DMS migration statistics.

This demonstrated that the change made after the initial migration was processed by the ongoing DMS replication task.

### CDC Proof

The project evidence includes a screenshot showing:

* DMS task status
* Full Load + CDC mode
* Completed table migration
* CDC activity after the source change

Therefore, the lab demonstrates both:

```text
Full Load
    +
Change Data Capture
```

---

# 🧩 Challenges & Solutions

## 1. Local MySQL Was Not Directly Reachable from AWS

### Problem

The MySQL database was running on a Windows laptop behind a residential internet connection.

The connection was using CGNAT, so traditional router port forwarding could not provide a reliable inbound path from AWS to the laptop.

Initial DMS connectivity attempts to the laptop's local IP resulted in:

```text
dial tcp 192.168.1.101:3306: i/o timeout
```

### Solution

Tailscale was introduced to provide private connectivity between the Windows laptop and AWS.

An EC2 instance was used as a bridge between AWS DMS and the Tailscale network.

---

## 2. Tailscale Subnet Routing Was More Complicated Than Necessary

### Problem

An initial approach attempted to expose the laptop's local subnet through Tailscale subnet routing.

This introduced additional routing complexity.

### Solution

The design was simplified.

Instead of routing the entire `192.168.1.0/24` network, the EC2 instance became a dedicated TCP proxy.

The final design used:

```text
EC2 → socat → Tailscale IP → MySQL
```

This reduced t
