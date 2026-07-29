# Web Solution With WordPress (Two-Tier Architecture)

## Overview
This project demonstrates the implementation of a scalable, multi-tier web application architecture on AWS EC2 instances running RedHat Enterprise Linux 9 (RHEL 9). The application tier running Apache HTTP Server and PHP 8.2 is decoupled from the database tier running MariaDB. Physical storage volumes attached to both instances are managed dynamically using Logical Volume Manager (LVM) to support flexible, non-disruptive storage allocation.

---

## Environment & Infrastructure Specifications
The deployment spans two dedicated EC2 instances within a single AWS Virtual Private Cloud (VPC), communicating securely over private network interfaces.

| Resource Role | Instance Name | Public IPv4 | Private IPv4 | Storage Allocation | Platform / OS |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Web Server** | `web-server` | `13.217.194.94` | `172.31.29.169` | 3x 10GB EBS (`apps-lv`, `logs-lv`) | RHEL 9 |
| **Database Server** | `db-server` | `44.204.119.85` | `172.31.2.179` | 3x 10GB EBS (`db-lv`) | RHEL 9 |

---

## Step-by-Step Implementation Guide

### Step 1: Storage Layer & LVM Setup (Web Server)
1. Attached 3x 10GB EBS block devices (`nvme1n1`, `nvme2n1`, `nvme3n1`) to the `web-server` instance.
2. Initialized partitions across all 3 disks using `fdisk`.
3. Converted physical partitions into LVM Physical Volumes (`pvcreate`) and aggregated them into a Volume Group (`webdata-vg`).
4. Provisioned two Logical Volumes:
   * `apps-lv` (14GB) — Formatted to `ext4` and mounted to `/var/www/html` for application files.
   * `logs-lv` (14GB) — Formatted to `ext4` and mounted to `/var/log` for system and HTTP log persistence.
5. Backed up existing `/var/log` contents using `rsync` prior to mounting, restored data post-mount, and configured persistent mounts in `/etc/fstab` using device `UUID`s.

### Step 2: Storage Layer & LVM Setup (Database Server)
1. Attached 3x 10GB EBS volumes to the `db-server` instance and partitioned each disk.
2. Created Physical Volumes and configured a Volume Group named `database-vg`.
3. Created a single 20GB Logical Volume (`db-lv`), formatted it with `ext4`, and mounted it to `/db`.
4. Configured automated mounting via `/etc/fstab` verified through `mount -a`.

### Step 3: Web Server Application Environment
* Installed Apache (`httpd`) and enabled modern PHP 8.2 runtime dependencies using Remi and EPEL repositories:

```
sudo dnf module enable php:remi-8.2 -y
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y
```
* Started and enabled php-fpm and httpd services.

* Downloaded, extracted, and structured WordPress source files inside /var/www/html/.

### Step 4: Database Provisioning & Network Binding

* Installed and secured mariadb-server on the Database instance (mysql_secure_installation).

* Created the application database and scoped administrative privileges exclusively to the Web Server's private IP address:

```
CREATE DATABASE wp_production_db;
CREATE USER 'wp_admin_user'@'172.31.29.169' IDENTIFIED BY 'Secure@wp2026';
GRANT ALL PRIVILEGES ON wp_production_db.* TO 'wp_admin_user'@'172.31.29.169';
FLUSH PRIVILEGES;
```
Bound MariaDB strictly to the server's private network interface (172.31.2.179) in /etc/my.cnf.d/mariadb-server.cnf.

Step 5: Integration & Verification

* Configured AWS Security Group inbound rules on the Database instance to allow TCP traffic over Port 3306 strictly from 172.31.29.169/32.

* Verified remote database connectivity from the Web Server using the mariadb client.

* Configured /var/www/html/wp-config.php with database credentials and the DB Server private IP (172.31.2.179).

*Completed the web-based WordPress setup wizard via web browser.

## Challenges EncounteredS

1\. Disk Partition Standard Alignment (MBR vs. GPT)

* **The Problem:** When I first ran fdisk, I forgot to add the -g flag, so it created an old MBR disk format instead of the modern GPT format.

* **The Solution:** I unmounted the volumes on both servers, re-run fdisk with -g to partition them as GPT properly, and then remount everything to continue.

2\. WordPress Routing & Directory Path

* **The Problem:** When i tried to access `http://<Web-Server-IP>/wordpress`, it gave a 404 Not Found error because all the extracted WordPress files were placed directly inside the root web directory (`/var/www/html/`) instead of its own dedicated application folder.

* **The Solution:** I created a dedicated `wordpress` sub-folder under `/var/www/html/`, moved all extracted core application files inside it, and set the appropriate Apache user permissions (`chown -R apache:apache /var/www/html/wordpress`) so the site loaded cleanly at `/wordpress`

## Key Takeaways

Decoupled Architecture: Keeping your website, storage, and database on separate servers means if one crashes, the whole system doesn't go down—and you can upgrade or expand each part independently whenever you need to.

Flexible Storage (LVM): Using LVM lets you easily add or increase storage space whenever you run out, without having to restart your servers or turn off your website.
