## Reference

[How to install PostgreSQL on Ubuntu 22.04 quickstart](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-22-04-quickstart)
[How to secure PostgreSQL against automated attacks](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-against-automated-attacks)

## Install
```bash
# update package list
sudo apt update

# install postgres and some extra useful functionality
sudo apt install postgresql postgresql-contrib
```

## Create database user
Login to a postgres session
```bash
sudo -u postgres psql
```
Create a new database for the Wagtail app to use
```sql
CREATE DATABASE my_database;
```
Create a new role (database user) so that the Wagtail app can connect and login with a password
```sql
CREATE ROLE my_user LOGIN PASSWORD 's00p3r_s3cr3t_p@$$w0rd';
```
Apply [Django's Postgres configuration recommendations](https://docs.djangoproject.com/en/4.1/ref/databases/#optimizing-postgresql-s-configuration) to our new database user
```sql
ALTER ROLE my_user SET client_encoding TO 'utf8';
ALTER ROLE my_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE my_user SET timezone TO 'UTC';
```
Grant all database [privileges](https://www.postgresql.org/docs/14/ddl-priv.html) to our new database user so it can do necessary admin actions (like creating tables)
```sql
GRANT ALL PRIVILEGES ON DATABASE my_database TO my_user;
```

## Basic security setup

### Configure UFW firewall (optional)
_If you'd prefer, you can use a [Digital Ocean Cloud Firewall](https://docs.digitalocean.com/tutorials/recommended-droplet-setup/#step-3-create-a-cloud-firewall) instead of UFW. To avoide confusing rules, it is best practice to use just one firewall (either UFW or Could Firewall)._

Follow [these steps](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04#step-4-setting-up-a-firewall) for initial UFW setup 

Then allow access to the port `5432` (the default port for Postgres) for the IP address of the client (in this case, the server hosting the Wagtail app).
```bash
sudo ufw allow from <wagtail_server_ip> to any port 5432
```

Check to make sure the new rule was added
```bash
sudo ufw status

# expected output (wish OpenSSH already allowed)
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
5432                       ALLOW       wagtail_server_ip
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

### Configure allowed hosts

Use nano to edit the `pg_hba.conf` (host-based authentication) file. 
```
sudo nano /etc/postgresql/<postgres_version_number>/main/pg_hba.conf
```

Just below the commented lines that describe how to allow non-local connections, add a new `host` line.
```
host    my_database   my_user   wagtail_server_ip/32    scram-sha-256
```
*See the top of the `pg_hba.conf` for a synopsis of the syntax and keywords found in the file.*

### Configure listening addresses
In the same directory as the `pg_hba.conf` file, edit `postgresql.conf`
```bash
sudo nano /etc/postgresql/<postgres_version_number>/main/postgresql.conf
```
Find the line that says `listen_addresses` and edit it to include the public IP address of the server hosting this Postgres instance
```bash
# 'localhost' is optional (I'm not sure if it's a good, neutral, or bad idea to include it)
listen_addresses = 'localhost,this_servers_ip'
```

### Restart PostgreSQL

Once you've edited `pg_hba.conf` and `postgresql.conf` as described above, restart Postgres so that your changes take effect.
```bash
sudo systemctl restart postgresql
```

Verify a successful restart with
```bash
sudo systemctl status postgresql

# Expected output:
# ...
# Aug 14 00:43:28 stbcms-db systemd[1]: Starting PostgreSQL RDBMS...
# Aug 14 00:43:28 stbcms-db systemd[1]: Finished PostgreSQL RDBMS.
```


## Require SSL Connections