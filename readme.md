# MySQL Replication Setup Guide: Binlog & GTID Methods

## 1. System Environment Preparation
Execute to remove OS security restrictions and release port bindings.

```bash
# Disable AppArmor confinement for MySQL
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld

# Disable system-wide MySQL service
sudo systemctl stop mysql

# Clean existing MySQL processes
killall -9 mysqld
```

## 2. Directory and Data Initialization
Create isolated data directories and perform insecure initialization.

```bash
# Cleanup and directory creation
rm -rf mysql-master mysql-slave
mkdir -p mysql-master mysql-slave

# Initialize Master data directory
mysqld --no-defaults --initialize-insecure --user=$(whoami) --datadir="$PWD/mysql-master"

# Initialize Slave data directory
mysqld --no-defaults --initialize-insecure --user=$(whoami) --datadir="$PWD/mysql-slave"
```

---

## Method A: Traditional Binlog Replication
Uses specific file names and byte offsets to track replication progress.

### 1. Instance Startup (Binlog Mode)
```bash
# Start Master
mysqld --no-defaults --user=$(whoami) --datadir="$PWD/mysql-master" --port=3306 --server-id=1 --mysqlx=OFF --log-bin="$PWD/mysql-master/mysql-bin" --socket="$PWD/mysql-master/mysql.sock" --pid-file="$PWD/mysql-master/mysql.pid" --lc-messages-dir=/usr/share/mysql --log-error="$PWD/mysql-master/error.log" &

# Start Slave
mysqld --no-defaults --user=$(whoami) --datadir="$PWD/mysql-slave" --port=3307 --server-id=2 --mysqlx=OFF --socket="$PWD/mysql-slave/mysql.sock" --pid-file="$PWD/mysql-slave/mysql.pid" --lc-messages-dir=/usr/share/mysql --log-error="$PWD/mysql-slave/error.log" &
```

### 2. Configuration
**On Master:**
```sql
CREATE USER 'replica_user'@'127.0.0.1' IDENTIFIED WITH mysql_native_password BY 'replica_password';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'127.0.0.1';
FLUSH PRIVILEGES;
SHOW MASTER STATUS; -- Record File and Position
```

**On Slave:**
```sql
CHANGE MASTER TO
    MASTER_HOST='127.0.0.1',
    MASTER_USER='replica_user',
    MASTER_PASSWORD='replica_password',
    MASTER_PORT=3306,
    MASTER_LOG_FILE='[FILE_FROM_MASTER]',
    MASTER_LOG_POS=[POS_FROM_MASTER];
START SLAVE;
```

---

## Method B: GTID-Based Replication
Uses Global Transaction IDs for automatic synchronization and simplified failover.

### 1. Instance Startup (GTID Mode)
```bash
# Start Master with GTID
mysqld --no-defaults --user=$(whoami) --datadir="$PWD/mysql-master" --port=3306 --server-id=1 --mysqlx=OFF --log-bin="$PWD/mysql-master/mysql-bin" --socket="$PWD/mysql-master/mysql.sock" --pid-file="$PWD/mysql-master/mysql.pid" --lc-messages-dir=/usr/share/mysql --gtid-mode=ON --enforce-gtid-consistency=ON --log-error="$PWD/mysql-master/error.log" &

# Start Slave with GTID
mysqld --no-defaults --user=$(whoami) --datadir="$PWD/mysql-slave" --port=3307 --server-id=2 --mysqlx=OFF --socket="$PWD/mysql-slave/mysql.sock" --pid-file="$PWD/mysql-slave/mysql.pid" --lc-messages-dir=/usr/share/mysql --gtid-mode=ON --enforce-gtid-consistency=ON --log-slave-updates=ON --log-error="$PWD/mysql-slave/error.log" &
```

### 2. Configuration
**On Master:**
Same as Method A (Ensure `replica_user` exists).

**On Slave:**
```sql
STOP SLAVE;
CHANGE MASTER TO
    MASTER_HOST='127.0.0.1',
    MASTER_USER='replica_user',
    MASTER_PASSWORD='replica_password',
    MASTER_PORT=3306,
    MASTER_AUTO_POSITION = 1;
START SLAVE;
```

---

## 3. Verification and Final Output
Confirm data propagation by checking the Slave instance.

```sql
SHOW SLAVE STATUS\G
-- Ensure Slave_IO_Running and Slave_SQL_Running are both 'Yes'
```
Master:
![Master Status](Screenshot%20from%202026-03-27%2014-36-53.png)

Slave:

![Final Output](Screenshot%20from%202026-03-27%2014-37-27.png)

## 4. Troubleshooting
If the SQL thread stops due to a duplicate key or existing user (Error 1396):

```sql
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; -- Only for Binlog Mode
START SLAVE;
```
*Note: For GTID mode, skip transactions by injecting an empty transaction with the offending GTID.*