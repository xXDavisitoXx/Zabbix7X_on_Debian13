# Zabbix7 on Debian13

## Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```
Configure the database directory (a separate disk from the system is recommended)

```bash
sudo systemctl stop postgresql
sudo rsync -av /var/lib/postgresql/17/main/ /mnt/database/postgresql/17/main/
sudo chown -R postgres:postgres /mnt/database/postgresql/17/main
```
Edit data directory
```bash
 sudo nano /etc/postgresql/18/main/postgresql.conf
```
```bash
data_directory = '/mnt/database/postgresql/17/main'  # use data in another directory
```
```bash
sudo systemctl start postgresql
```
```bash
sudo pg_lsclusters
```
```bash
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /mnt/database/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

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
