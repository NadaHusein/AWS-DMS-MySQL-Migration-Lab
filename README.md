# AWS DMS Migration — Local MySQL to Amazon RDS

## Project Overview

This project demonstrates how to migrate a MySQL database running locally on a Windows laptop to Amazon RDS for MySQL using **AWS Database Migration Service (AWS DMS)**.

The migration architecture is designed to support both:

* **Full Load** — migrate the existing database contents.
* **Change Data Capture (CDC)** — continuously replicate subsequent changes from the source database.

### Target Architecture

```text
┌──────────────────────┐
│   Local Windows PC   │
│                      │
│   MySQL 8.0          │
│   Database: shop     │
│   192.168.1.101:3306 │
└──────────┬───────────┘
           │
           │ Tailscale VPN
           │
           ▼
┌──────────────────────┐
│    EC2 Bridge        │
│  Amazon Linux 2023   │
│  t3.micro            │
│  172.31.5.179        │
└──────────┬───────────┘
           │
           │ AWS VPC
           ▼
┌──────────────────────┐
│   AWS DMS            │
│   Replication        │
│   Instance           │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Amazon RDS         │
│   MySQL              │
└──────────────────────┘
```

---

## 1. Local MySQL Source

MySQL 8.0 was installed on the Windows laptop and configured as the source database.

A sample e-commerce database named `shop` was created.

### Database Schema

The database contains three related tables:

```text
customers
products
orders
```

The `orders` table contains foreign keys referencing:

* `customers.customer_id`
* `products.product_id`

### Initial Data

| Table     | Rows |
| --------- | ---: |
| customers |    5 |
| products  |    5 |
| orders    |    6 |

The source data was verified locally using SQL queries before starting the migration.

---

## 2. Source Database CDC Configuration

Because the migration will use **Full Load + CDC**, the MySQL binary log configuration was checked.

The following settings were verified:

```text
log_bin          = ON
binlog_format    = ROW
binlog_row_image = FULL
```

These settings provide the required binary-log information for change data capture.

---

## 3. Network Connectivity Challenge

The source database is running on a laptop inside a home network.

The laptop has a private IP:

```text
192.168.1.101
```

The MySQL service listens on:

```text
3306
```

Initially, direct connectivity from AWS to:

```text
192.168.1.101:3306
```

was not possible.

The home Internet connection was found to use **Carrier-Grade NAT (CGNAT)**. Therefore, traditional Internet port forwarding could not provide a reliable inbound path from AWS to the laptop.

Exposing MySQL directly to the public Internet was also intentionally avoided.

---

## 4. Tailscale VPN / Network Bridge

To establish private connectivity without exposing MySQL publicly, **Tailscale** was deployed between the laptop and an EC2 instance.

### Tailscale Addresses

```text
Laptop
100.97.250.23

EC2
100.79.148.113
```

The EC2 instance was created specifically as a network bridge.

### EC2 Configuration

```text
Name:        dms-vpn-bridge
Instance:    i-0897b7e3e614464aa
OS:          Amazon Linux 2023
Type:        t3.micro
Private IP:  172.31.5.179
```

Tailscale was installed on both the laptop and EC2 instance.

---

## 5. IP Forwarding

The EC2 instance was configured to forward IPv4 traffic:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Validation:

```text
net.ipv4.ip_forward = 1
```

Source/destination checks were also disabled on the EC2 instance because it is being used to forward traffic rather than only receive traffic addressed directly to itself.

---

## 6. Tailscale Subnet Routing

The laptop was configured to advertise its local network through Tailscale:

```text
192.168.1.0/24
```

The route was approved in the Tailscale administration console.

The EC2 instance was configured to accept advertised routes:

```bash
sudo tailscale set --accept-routes=true
```

---

## 7. AWS VPC Routing

An AWS VPC route was added so that traffic destined for the local network is forwarded to the EC2 bridge.

```text
Destination:
192.168.1.0/24

Target:
dms-vpn-bridge EC2
```

This allows the AWS-side network to use the EC2 instance as the path toward the laptop network.

---

## 8. Connectivity Validation

Connectivity was tested progressively from the EC2 instance.

### Test 1 — Tailscale connectivity

The EC2 instance successfully reached the laptop's Tailscale IP:

```bash
ping -c 4 100.97.250.23
```

Result:

```text
4 packets transmitted
4 received
0% packet loss
```

### Test 2 — MySQL over Tailscale

```bash
nc -zv 100.97.250.23 3306
```

Result:

```text
Connected to 100.97.250.23:3306
```

### Test 3 — MySQL using the laptop's LAN IP

After configuring subnet routing and AWS networking:

```bash
nc -zv 192.168.1.101 3306
```

Result:

```text
Connected to 192.168.1.101:3306
```

### Validation Result

The AWS EC2 bridge can now reach the local MySQL database through the following path:

```text
AWS EC2
   ↓
AWS VPC routing
   ↓
Tailscale
   ↓
Laptop
   ↓
MySQL :3306
```

This establishes the network connectivity required for the next stage: configuring AWS DMS.

---

## Current Status

### Completed

* [x] Install MySQL 8.0 locally
* [x] Create `shop` database
* [x] Create source tables
* [x] Insert sample data
* [x] Verify source data
* [x] Verify MySQL binary logging / CDC configuration
* [x] Identify CGNAT limitation
* [x] Create EC2 bridge
* [x] Install Tailscale
* [x] Establish EC2 ↔ laptop connectivity
* [x] Enable IP forwarding
* [x] Configure Tailscale subnet routing
* [x] Configure AWS VPC routing
* [x] Validate EC2 → laptop MySQL connectivity

### Next Steps

* [ ] Create AWS DMS replication instance
* [ ] Create MySQL source endpoint
* [ ] Create Amazon RDS MySQL target
* [ ] Create target endpoint
* [ ] Test DMS endpoint connectivity
* [ ] Create Full Load + CDC replication task
* [ ] Validate migrated data
* [ ] Validate CDC using INSERT / UPDATE / DELETE
* [ ] Capture final evidence/screenshots
* [ ] Delete billable AWS resources after the lab
