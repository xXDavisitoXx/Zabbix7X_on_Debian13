# Zabbix7 on Debian13

## Install Zabbix repository

```bash
# wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb
# dpkg -i zabbix-release_latest_7.4+debian13_all.deb
# apt update
```
---
## Install Zabbix server, frontend, agent

```bash
apt install zabbix-server-pgsql zabbix-frontend-php php8.4-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

## Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```
Configure the database directory (a separate disk from the system is recommended)

```bash
sudo systemctl stop postgresql
sudo mkdir -p /mnt/database/postgresql/17/main/ # Or the custom path where you want the database to live
sudo rsync -av /var/lib/postgresql/17/main/ /mnt/database/postgresql/17/main/
sudo chown -R postgres:postgres /mnt/database/postgresql/17/main
```
Edit data directory
```bash
 sudo nano /etc/postgresql/17/main/postgresql.conf
```
```bash
data_directory = '/mnt/database/postgresql/17/main'  # use data in another directory
```
Restart database service:
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
Create user zabbixdb
```bash
sudo -u postgres createuser --pwprompt zabbixdb
```
Create database  dbzabbix:
```bash
sudo -u postgres createdb -O zabbixdb dbzabbix
```

## Install TimeScaleDB
Download the repository keys:
```bash
sudo curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey \
| gpg --dearmor -o /usr/share/keyrings/timescaledb.gpg
```
Install the repository of TimeScaleDB:
```bash
sudo echo "deb [signed-by=/usr/share/keyrings/timescaledb.gpg] \
https://packagecloud.io/timescale/timescaledb/debian/ trixie main" \
> /etc/apt/sources.list.d/timescaledb.list
```
Refresh APT:
```bash
sudo apt update
```
Install TimeScaleDB from APT:
```bash
apt install timescaledb-2-postgresql-17
```
Configure TimeScale for PostgreSQL:
```bash
sudo timescaledb-tune --quiet --yes
```
Activate TimeScaleDB on dbzabbix:
```bash
sudo -u postgres psql -d dbzabbix -c "CREATE EXTENSION IF NOT EXISTS timescaledb;"
```
Import schema database:
```bash
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbixdb psql -d dbzabbix
```
Restart database service:
```bash
systemctl restart postgresql
```






