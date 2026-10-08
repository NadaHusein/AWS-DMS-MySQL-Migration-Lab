# AWS DMS MySQL Migration Lab

## Migrating a Local MySQL Database to Amazon RDS using AWS Database Migration Service

### 1. Project Objective

The objective of this lab was to simulate an on-premises MySQL environment and migrate the database to Amazon RDS using AWS Database Migration Service (AWS DMS).

The source database was hosted locally on a Windows laptop, while AWS DMS and the target database were hosted in AWS.

The migration design was intended to support:

* Full Load migration
* Change Data Capture (CDC)
* MySQL → MySQL migration
* Secure connectivity without exposing MySQL port 3306 directly to the public Internet

---

## 2. Architecture

```text
                    AWS
┌──────────────────────────────────────────────────────┐
│                                                      │
│  AWS DMS                                             │
│  shop-dms-replication                                │
│  172.31.27.19                                        │
│        │                                             │
│        │ TCP 3306                                    │
│        ▼                                             │
│  EC2 Bridge / Proxy                                  │
│  dms-vpn-bridge                                      │
│  172.31.5.179                                        │
│        │                                             │
└────────┼─────────────────────────────────────────────┘
         │
         │ Tailscale
         ▼
┌──────────────────────────────┐
│ Local Windows Laptop         │
│ Tailscale: 100.97.250.23     │
│ LAN: 192.168.1.101           │
│                              │
│ MySQL 8.0                    │
│ Database: shop               │
│ Port: 3306                   │
└──────────────────────────────┘
```

The EC2 instance acts as a TCP bridge/proxy between AWS DMS and the local MySQL database.

---

# 3. Source Database

MySQL 8.0 was installed locally on Windows.

Database:

```text
shop
```

Tables:

```text
customers
products
orders
```

Initial data:

| Table     | Rows |
| --------- | ---: |
| customers |    5 |
| products  |    5 |
| orders    |    6 |

The source data was verified using SQL queries before starting the migration.

---

# 4. MySQL CDC Configuration

The source MySQL instance was checked for AWS DMS CDC requirements.

The following settings were verified:

```text
log_bin = ON
binlog_format = ROW
binlog_row_image = FULL
```

These settings are appropriate for MySQL binary-log based Change Data Capture.

---

# 5. Initial Connectivity Approach

The first approach was to allow AWS DMS to connect directly to the laptop's private LAN address:

```text
192.168.1.101:3306
```

However, the laptop was behind a home router using CGNAT.

The router WAN address was in the:

```text
100.64.0.0/10
```

CGNAT address range.

Therefore, traditional Internet port forwarding could not provide a reliable inbound path from AWS to the laptop.

Exposing MySQL directly to the public Internet was also intentionally avoided.

---

# 6. Tailscale-Based Connectivity

Tailscale was selected to provide connectivity between the AWS bridge instance and the local laptop without requiring a public inbound connection to the home network.

Tailscale devices:

```text
EC2:
100.79.148.113

Windows laptop:
100.97.250.23
```

The laptop advertised the local subnet:

```text
192.168.1.0/24
```

The route was approved in the Tailscale administration console.

IP forwarding was enabled on the EC2 instance and Windows laptop.

The EC2 instance also accepted the advertised route.

---

# 7. AWS VPC Routing

The AWS route table was configured with:

```text
Destination:
192.168.1.0/24

Target:
EC2 bridge ENI
```

The EC2 instance had Source/Destination Check disabled because it was being used as a network bridge.

The DMS subnet therefore had a route toward the local LAN through the EC2 bridge.

---

# 8. First Major Troubleshooting Challenge

The first AWS DMS source endpoint connection test failed with:

```text
Can't connect to MySQL server on '192.168.1.101' (110)

dial tcp 192.168.1.101:3306:
i/o timeout
```

This indicated a network-level connectivity problem rather than a MySQL authentication problem.

---

# 9. Packet-Level Investigation

`tcpdump` was used on the EC2 bridge.

Traffic from the DMS replication instance was observed:

```text
172.31.27.19 → 192.168.1.101:3306
```

Repeated TCP SYN packets were observed.

However, no SYN-ACK response was returned.

This proved that:

* DMS traffic reached the EC2 instance.
* The AWS route was being used.
* The problem was occurring further along the path.

---

# 10. Testing Tailscale Connectivity

Tailscale connectivity between EC2 and Windows was verified.

The EC2 instance successfully connected to the laptop's Tailscale IP:

```text
100.97.250.23:3306
```

Example:

```text
Ncat: Connected to 100.97.250.23:3306.
```

This proved that:

```text
EC2 → Tailscale → Windows → MySQL
```

worked successfully.

However, forwarded DMS traffic still failed to reach Windows.

Windows `pktmon` did not show the DMS packets arriving on port 3306.

---

# 11. Decision: Use EC2 as a TCP Proxy

Instead of continuing to troubleshoot subnet-router forwarding, the architecture was simplified.

EC2 was configured as a TCP proxy using `socat`.

The resulting path became:

```text
DMS
 │
 │ 172.31.5.179:3306
 ▼
EC2
 │
 │ socat
 ▼
100.97.250.23:3306
 │
 ▼
Local MySQL
```

This removed the requirement for DMS to directly route to the home LAN.

DMS only needs normal AWS VPC connectivity to the EC2 instance.

---

# 12. Security Group Configuration

The EC2 security group allows MySQL traffic on port 3306 from the DMS security group.

Conceptually:

```text
TCP 3306
Source:
shop-dms-sg
```

This avoids allowing MySQL traffic from the entire Internet.

SSH access was temporarily opened during troubleshooting and should be restricted or removed after the lab.

---

# 13. Second Major Troubleshooting Challenge

After switching the DMS endpoint to:

```text
172.31.5.179:3306
```

the DMS connection initially returned:

```text
connection refused
```

This occurred because the `socat` process was running in the EC2 terminal.

When the terminal/session ended, the foreground `socat` process stopped.

The solution was to run `socat` as a persistent systemd service.

---

# 14. Third Major Troubleshooting Challenge — MySQL Host Authorization

Once `socat` was running, DMS successfully reached MySQL.

The error changed to:

```text
Host 'ip-172-31-5-179.tail7e6d47.ts.net.'
is not allowed to connect to this MySQL server
```

This was an important milestone because it proved that the network connection was now working.

The failure had moved from the networking layer to the MySQL authorization layer.

---

# 15. MySQL User Investigation

The existing MySQL users were checked:

```sql
SELECT user, host FROM mysql.user;
```

The result showed:

```text
root | localhost
```

Therefore, the `root` account was only configured for local connections.

The DMS connection was coming through the EC2/Tailscale path and was therefore not matching:

```text
root@localhost
```

---

# 16. Dedicated DMS User

Instead of modifying the root account, a dedicated migration user was created:

```sql
CREATE USER 'dms_user'@'%'
IDENTIFIED BY 'DmsLab@2026!';
```

Global privileges required for the migration/CDC configuration were granted separately from database-level privileges.

Database-level permissions were granted specifically for:

```text
shop.*
```

This follows the principle of using a dedicated migration account rather than using the MySQL root account.

---

# 17. Final Successful Connectivity Test

The DMS source endpoint was configured with:

```text
Server:
172.31.5.179

Port:
3306

Username:
dms_user

SSL:
none
```

The AWS DMS endpoint connection test then returned:

```text
Successful
```

This confirmed the complete connectivity chain:

```text
AWS DMS
   │
   ▼
EC2 172.31.5.179
   │
   ▼
socat TCP proxy
   │
   ▼
Tailscale
   │
   ▼
Windows 100.97.250.23
   │
   ▼
MySQL 8.0
```

---

# 18. Key Troubleshooting Lessons

### Lesson 1 — Separate network failures from authentication failures

The error:

```text
i/o timeout
```

indicated a connectivity/routing problem.

Later:

```text
Host is not allowed to connect
```

proved that network connectivity had been established and the problem had moved to MySQL authorization.

---

### Lesson 2 — Test each network layer independently

Useful tests included:

```bash
nc -zv 100.97.250.23 3306
```

and:

```bash
nc -zv 172.31.5.179 3306
```

These tests verified TCP reachability without requiring MySQL authentication.

---

### Lesson 3 — Packet capture can identify where traffic stops

`tcpdump` showed the DMS SYN packets reaching EC2.

`pktmon` on Windows showed that the packets were not reaching the Windows host.

This narrowed the problem to the forwarding path.

---

### Lesson 4 — CGNAT changes the connectivity design

A local machine behind CGNAT cannot reliably be treated as an Internet-accessible server using ordinary port forwarding.

A VPN/overlay network or another outbound connectivity mechanism is required.

---

### Lesson 5 — Use dedicated database users

Using:

```text
root
```

for a migration is unnecessary.

A dedicated DMS account provides a cleaner security boundary and makes troubleshooting easier.

---

# 19. Final Migration Plan

The remaining migration steps are:

1. Create the target Amazon RDS MySQL database.
2. Create the DMS target endpoint.
3. Test the target endpoint.
4. Create a DMS migration task.
5. Run Full Load.
6. Verify the migrated tables and rows.
7. Test CDC by inserting/updating/deleting source data.
8. Verify the changes appear on RDS.
9. Capture final evidence/screenshots.
10. Remove paid AWS resources after the lab.

---

# 20. Evidence to Capture

Recommended evidence:

* Local MySQL database and tables
* Source row counts
* MySQL CDC configuration
* EC2 bridge configuration
* Tailscale connected devices
* AWS VPC route
* DMS replication instance
* Successful source endpoint connection
* Target RDS database
* Successful migration task
* Migrated row counts
* Successful CDC test

---

# 21. Final Outcome

The lab successfully established connectivity between a locally hosted MySQL database and AWS DMS despite the source environment being behind CGNAT.

The final design used:

* MySQL 8.0
* AWS DMS
* Amazon EC2
* Tailscale
* `socat`
* AWS VPC routing
* Security Groups
* MySQL binary logging
* Dedicated DMS database credentials

The project also demonstrated practical troubleshooting across multiple layers:

```text
Application
    ↓
Database authentication
    ↓
TCP connectivity
    ↓
Tailscale
    ↓
EC2 forwarding/proxy
    ↓
AWS VPC routing
    ↓
Security Groups
```

This makes the lab more representative of a real infrastructure troubleshooting scenario than a simple click-through DMS migration.
