# MySQL Read Replica Setup Guide

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

## 3. Instance Startup
Launch the background daemons using unique ports, sockets, and server IDs.

### Start Master (Port 3306)
```bash
mysqld --no-defaults --user=$(whoami) \
  --datadir="$PWD/mysql-master" \
  --port=3306 \
  --server-id=1 \
  --mysqlx=OFF \
  --log-bin="$PWD/mysql-master/mysql-bin" \
  --socket="$PWD/mysql-master/mysql.sock" \
  --pid-file="$PWD/mysql-master/mysql.pid" \
  --lc-messages-dir=/usr/share/mysql \
  --log-error="$PWD/mysql-master/error.log" &
```

### Start Slave (Port 3307)
```bash
mysqld --no-defaults --user=$(whoami) \
  --datadir="$PWD/mysql-slave" \
  --port=3307 \
  --server-id=2 \
  --mysqlx=OFF \
  --socket="$PWD/mysql-slave/mysql.sock" \
  --pid-file="$PWD/mysql-slave/mysql.pid" \
  --lc-messages-dir=/usr/share/mysql \
  --log-error="$PWD/mysql-slave/error.log" &
```

## 4. Replication Configuration

### Master Configuration
Connect via Master socket:
`mysql -u root --socket="$PWD/mysql-master/mysql.sock"`

```sql
-- Use mysql_native_password for local socket compatibility
CREATE USER 'replica_user'@'127.0.0.1' IDENTIFIED WITH mysql_native_password BY 'replica_password';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'127.0.0.1';
FLUSH PRIVILEGES;

-- Capture File and Position values
SHOW MASTER STATUS;
```

### Slave Configuration
Connect via Slave socket:
`mysql -u root --socket="$PWD/mysql-slave/mysql.sock"`

```sql
CHANGE MASTER TO
    MASTER_HOST='127.0.0.1',
    MASTER_USER='replica_user',
    MASTER_PASSWORD='replica_password',
    MASTER_PORT=3306,
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=889;

START SLAVE;
```

## 5. Troubleshooting
If SHOW SLAVE STATUS reports Error 1396 (User already exists), skip the conflicting transaction:

```sql
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
```

## 6. Verification and Final Output
Confirm data propagation by checking the Slave instance.

![Master Setup](Screenshot%20from%202026-03-27%2014-36-53.png)
![Final Output](Screenshot%20from%202026-03-27%2014-37-27.png)