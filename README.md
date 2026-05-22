# Zabbix7 on Debian13
## Install software
### Install system packages:
As root user:
```bash
apt update
apt install rsync sudo gnupg curl
```
Add our system user to sudo:
```bash
usermod -aG sudo YOUR_USER
```
Exit the root session, from now on we will use our regular system user
---
### Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```
### Install Zabbix repository:
```bash
sudo wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb
sudo dpkg -i zabbix-release_latest_7.4+debian13_all.deb
```
Refresh APT:
```bash
sudo apt update
```
### Install Zabbix server, frontend, Zabbix agent, Nginx 
```bash
sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.4-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent nginx php-fpm
```
## Configure PostgreSQL 
### Configure the database directory (a separate disk from the system is recommended)
```bash
sudo systemctl stop postgresql
sudo mkdir -p /mnt/database/postgresql/17/main/ # Or the custom path where you want the database to live
sudo rsync -av /var/lib/postgresql/17/main/ /mnt/database/postgresql/17/main/
sudo chown -R postgres:postgres /mnt/database/postgresql
```
Edit data directory
```bash
 sudo nano /etc/postgresql/17/main/postgresql.conf
```
```bash
data_directory = '/mnt/database/postgresql/17/main'  # use data in another directory
```
Start database service:
```bash
sudo systemctl start postgresql
```
Test new config:
```bash
sudo pg_lsclusters
```
```bash
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /mnt/database/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```
### Configure database acces:
Create user zabbixdb
```bash
sudo -u postgres createuser --pwprompt zabbixdb
```
⚠️ Save the password you set for later use

Create database  dbzabbix:
```bash
sudo -u postgres createdb -O zabbixdb dbzabbix
```

### Install TimeScaleDB plugin:
Download the repository keys:
```bash
sudo curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey \
| sudo gpg --dearmor -o /usr/share/keyrings/timescaledb.gpg
```
Install the repository of TimeScaleDB:
```bash
echo "deb [signed-by=/usr/share/keyrings/timescaledb.gpg] https://packagecloud.io/timescale/timescaledb/debian/ trixie main" \
| sudo tee /etc/apt/sources.list.d/timescaledb.list > /dev/null
```
Refresh APT:
```bash
sudo apt update
```
Install TimeScaleDB from APT:
```bash
sudo apt install timescaledb-2-postgresql-17=2.26.4~debian13-1709 timescaledb-2-loader-postgresql-17=2.26.4~debian13-1709
```
Configure TimeScale for PostgreSQL:
```bash
sudo timescaledb-tune --quiet --yes
```
Restart database service:
```bash
sudo systemctl restart postgresql
```
Activate TimeScaleDB on dbzabbix:
```bash
sudo -u postgres psql -d dbzabbix -c "CREATE EXTENSION IF NOT EXISTS timescaledb;"
```
Import schema database:
```bash
sudo zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | psql -U zabbixdb -d dbzabbix -h localhost
```
Execute schema optimization TimeScaleDB:
```bash
sudo -u postgres psql -d dbzabbix -f /usr/share/zabbix/sql-scripts/postgresql/timescaledb/schema.sql
```
Test TimeScaleDB:
```bash
 sudo -u postgres psql -d dbzabbix -c "SELECT hypertable_name FROM timescaledb_information.hypertables;"
```
Correct result:
```bash
  hypertable_name
-----------------
 history
 history_uint
 history_log
 history_text
 history_str
 history_bin
 auditlog
 trends
 trends_uint
(9 filas)
```

## Configure Zabbix server:
### Skip TimescaleDB version restriction:
Uncomment and set the AllowUnsupportedDBVersions parameter to 1:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
```bash
### Option: AllowUnsupportedDBVersions
#       Allow server to work with unsupported database versions.
#       0 - do not allow
#       1 - allow
#
# Mandatory: no
# Default:
AllowUnsupportedDBVersions=1
```
### Configure the database credentials for zabbix server:
Change parameters, in this file, we will add the password for the PostgreSQL user zabbixdb that we created earlier
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
```bash
### Option: DBName
#       Database name.
#       If the Net Service Name connection method is used to connect to Oracle database, specify the service name from
#       the tnsnames.ora file or set to empty string; also see the TWO_TASK environment variable if DBName is set to
#       empty string.
#
# Mandatory: yes
# Default:
# DBName=

DBName=dbzabbix

### Option: DBSchema
#       Schema name. Used for PostgreSQL.
#
# Mandatory: no
# Default:
# DBSchema=

### Option: DBUser
#       Database user.
#
# Mandatory: no
# Default:
# DBUser=

DBUser=zabbixdb

### Option: DBPassword
#       Database password.
#       Comment this line if no password is used.
#
# Mandatory: no
# Default:
DBPassword=YOUR_PASSWORD_DB_USER
```
Enable global scripts:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
```bash
### Option: EnableGlobalScripts
#    Enable global scripts on Zabbix server.
#       0 - disable
#       1 - enable
#
# Mandatory: no
# Default:
EnableGlobalScripts=1
# EnableGlobalScripts=0
```
### Configure PHP for Zabbix frontend:
Uncomment the listen and server_name lines with the parameters that will be used in the URL; this can be a DNS name or an IP address:
```bash
sudo nano /etc/zabbix/nginx.conf
```
```bash
server {
        listen          8080;
        server_name     192.168.1.100;
```
Restart web services:
```bash
sudo systemctl restart zabbix-server zabbix-agent nginx php8.4-fpm
```

### 
Enable services: 
```bash
sudo systemctl enable zabbix-server zabbix-agent nginx php8.4-fpm postgresql
```
