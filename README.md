# Disaster Recovery (DR) Setup for WordPress with Live Replication

This guide outlines how to set up a Disaster Recovery (DR) site for a WordPress website using live replication of files and MySQL database over a secure IPSec VPN tunnel.

## Overview

- **Primary Server**: Main WordPress site
- **DR Server**: Failover site
- **Security**: IPSec VPN tunnel using FortiGate Firewall
- **File Sync**: `rsync` over SSH tunnel
- **Database Replication**: MySQL master-slave replication
- **Failover Handling**: Manual or automated DNS switch

---

## 1. Set Up an IPSec VPN Tunnel and SSH Authentication

### IPSec VPN
Set up an IPSec VPN tunnel between the primary and DR servers using FortiGate Firewall.

### SSH Key-Based Authentication
On the Primary server:
```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id user@dr-server-private-ip
```
Test the connection:
```bash
ssh user@dr-server-ip
```

---

## 2. Live Replication of WordPress Files (rsync Over SSH Tunnel)

### Install rsync
```bash
sudo apt install rsync -y
```

### Sync Files
```bash
rsync -avz --delete -e "ssh -p 22" /var/www/html/ user@dr-server-ip:/var/www/html/
```

### Automate Sync via Cron Job
```bash
crontab -e
```
Add the following line to sync every 5 minutes:
```bash
*/5 * * * * rsync -avz --delete -e "ssh -p 22" /var/www/html/ user@dr-server-ip:/var/www/html/
```

---

## 3. Live MySQL Database Replication (Master-Slave Over SSH)

### Configure MySQL Master on Primary Server
Edit MySQL config:
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Modify:
```
[mysqld]
server-id=1
log_bin=/var/log/mysql/mysql-bin.log
binlog_do_db=wordpress_db  # Replace with your DB name
```
Restart MySQL:
```bash
sudo systemctl restart mysql
```

Create a replication user:
```sql
CREATE USER 'replica'@'dr-server-ip' IDENTIFIED BY 'securepassword';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'dr-server-ip';
FLUSH PRIVILEGES;
```
Get master status:
```sql
SHOW MASTER STATUS;
```
(Note down `File` and `Position` values.)

### Configure MySQL Slave on DR Server
Edit MySQL config:
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Modify:
```
[mysqld]
server-id=2
relay_log=/var/log/mysql/mysql-relay-bin.log
```
Restart MySQL:
```bash
sudo systemctl restart mysql
```

Set up slave replication:
```sql
CHANGE MASTER TO 
MASTER_HOST='primary-server-ip', 
MASTER_USER='replica', 
MASTER_PASSWORD='securepassword', 
MASTER_LOG_FILE='mysql-bin.000001',  -- Replace with master file
MASTER_LOG_POS=120;  -- Replace with master position
START SLAVE;
SHOW SLAVE STATUS\G;
```

---

## 4. Monitoring & Backup

### Check rsync Logs
```bash
tail -f /var/log/syslog | grep rsync
```

### Check MySQL Replication
```sql
SHOW SLAVE STATUS\G;
```
If `Slave_IO_Running` or `Slave_SQL_Running` shows `No`, fix it:
```sql
STOP SLAVE;
RESET SLAVE;
START SLAVE;
```

### Backup DR Database (Optional)
```bash
mysqldump -u root -p wordpress_db > /backup/wordpress_db_$(date +%F).sql
```

---

## ✅ Summary
✔ File replication with rsync over SSH  
✔ Live MySQL replication over IPSec tunnel  
✔ Regular backup & monitoring  

---

## 🛠 Troubleshooting

### Connection Refused (SHOW SLAVE STATUS\G;)
- Ensure `bind-address` in `/etc/mysql/mysql.conf.d/mysqld.cnf` allows the DR server (`0.0.0.0` or specific private IP).

### SSL Authentication Errors
Enable SSL on both master and slave, or disable it:
```sql
CHANGE MASTER TO MASTER_SSL=1;
START SLAVE;
SHOW SLAVE STATUS\G;
```

---

### FROM MASTER (TEST DB):
![Jenkins Job Configuration](https://github.com/ssanthosh2k3/DR-Setup-MYSQL/blob/main/Screenshot%20from%202025-02-13%2016-08-39.png)

### FROM SLAVE:
![Jenkins Job Configuration](https://github.com/ssanthosh2k3/DR-Setup-MYSQL/blob/main/Screenshot%20from%202025-02-13%2016-08-46.png)
![Jenkins Job Configuration](https://github.com/ssanthosh2k3/DR-Setup-MYSQL/blob/main/Screenshot%20from%202025-02-13%2016-09-31.png)


--------------------------------------



