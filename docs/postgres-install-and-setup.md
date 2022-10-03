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
_If you'd prefer, you can use a [Digital Ocean Cloud Firewall](https://docs.digitalocean.com/tutorials/recommended-droplet-setup/#step-3-create-a-cloud-firewall) instead of UFW. To avoid confusing or overlapping rules, it is best practice to use just one firewall (either UFW or Could Firewall)._

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
sudo nano /etc/postgresql/<postgres_version>/main/pg_hba.conf
```

Just below the commented lines that describe how to allow non-local connections, add a new `host` line.
```
host    my_database   my_user   wagtail_server_ip/32    scram-sha-256
```
*See the top of the `pg_hba.conf` for a synopsis of the syntax and keywords found in the file.*

### Configure listening addresses
In the same directory as the `pg_hba.conf` file, edit `postgresql.conf`
```bash
sudo nano /etc/postgresql/<postgres_version>/main/postgresql.conf
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

## Setup SSL

### Install Certbot

[Installing Certbot on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-use-certbot-standalone-mode-to-retrieve-let-s-encrypt-ssl-certificates-on-ubuntu-22-04#step-1-installing-certbot)
```bash
sudo snap install core

sudo snap refresh core

sudo apt remove certbot

sudo snap install --classic certbot
```

### Setup domain for database 
[Create a subdomain](https://docs.digitalocean.com/products/networking/dns/how-to/add-subdomain/) of your website domain and point it at the droplet hosting the database. Since we don't intend to share our database domain name publically, it's not a bad idea to use a random string generator to include some random characters in the domain name to make it harder to guess.

It might look something like
```
db-randomcharacterstirng.yourwebsite.com
```

_Below, I'll refer to this domain as `db.domain.com`_

### Install SSL certificate with Certbot and configure for Postgres

_The instructions below draw from the [Cerbot User Guide](https://eff-certbot.readthedocs.io/en/stable/using.html#user-guide) and [this blog post](https://loganmarchione.com/2020/10/securing-postgres-connections-using-lets-encrypt-certificates/)._

_Follow [these instructions](https://eff-certbot.readthedocs.io/en/stable/using.html#modifying-the-renewal-configuration-of-existing-certificates) if you need to **modify the renewal configuration for existing certificates**._

_Carefully follow these [certificate deletion instructions](https://eff-certbot.readthedocs.io/en/stable/using.html#safely-deleting-certificates) if you need to **delete a certificate** for some reason._


1. Edit your `postgresql.conf` file to ensure that SSL is enabled and the filepaths for SSL certs and keys are specified.
    ```bash
    # Use nano to edit the file
    sudo nano /etc/postgresql/<postgres_version>/main/postgresql.conf

    # Edit these lines
    ssl = on
    ssl_cert_file = '/etc/postgresql/<postgres_version>/main/fullchain.pem'
    ssl_key_file = '/etc/postgresql/<postgres_version>/main/privkey.pem'
    ```
2. Define a post-renew hook by creating a file called `setup-postgres` inside `/etc/letsencrypt/renewal-hooks/post/` and add the following contents
    ```bash
    #!/usr/bin/bash

    SOURCE=/etc/letsencrypt/live/db.domain.com
    DEST=/etc/postgresql/<postgres_version>/main

    cp $SOURCE/fullchain.pem $DEST/fullchain.pem
    cp $SOURCE/privkey.pem $DEST/privkey.pem
    chmod 600 $DEST/fullchain.pem $DEST/privkey.pem
    chown postgres:postgres $DEST/fullchain.pem $DEST/privkey.pem
    systemctl restart postgresql
    ```
    After [renewing a certificate](https://eff-certbot.readthedocs.io/en/stable/using.html#renewing-certificates), certbot will run this script which will copy the new certificate to the location we configured above for postgres to look.
3. Make your new post-renew hook executable
    ```bash
    sudo chmod +x /etc/letsencrypt/renewal-hooks/post/setup-postgres
    ```
4. Download and install a certificate with Certbot
    ```bash
    sudo certbot certonly \
      --standalone \
      --agree-tos \
      --eff-email \
      --email your@email.com \
      -d db.domain.com
    ```
5. Update the `host` entry in our `pg_hba.conf` file to be a `hostssl` entry
6. Update any connection strings for clients you want to use SSL to include `sslmode=verify-full` **and** `sslrootcert=/etc/ssl/certs/ca-certificates.crt`. In our case, the client is a wagtail app running in a docker container on another droplet.
    ```
    postgresql://db_user:password@db.domain.com:5432/db_name?sslmode=verify-full&sslrootcert=/etc/ssl/certs/ca-certificates.crt
    ```
    `sslrootcert` points to the file containing our CA's (Certificate Authority) root certificate. The Ubuntu package `ca-certificates` will contain our root cert and should be installed in our docker container already. You can verify that `ca-certificates` is installed and has the root certificates by running
    ```bash
    docker exec -it <container_name> 
    ```
    Refer to the [ca-certificates package documentation](https://ubuntu.com/server/docs/security-trust-store) for more.
7. Verify clients are connecting with SSL
    ```sql
    SELECT ssl.pid, usename, datname, ssl, ssl.version, ssl.cipher, ssl.bits, client_addr
    FROM pg_catalog.pg_stat_ssl ssl, pg_catalog.pg_stat_activity activity
    WHERE ssl.pid = activity.pid;
    ```
    expected output
    ```
    pid   |  usename  | datname | ssl | version |         cipher         | bits |  client_addr
    ------+-----------+---------+-----+---------+------------------------+------+-------------------
    51664 | my_app    | my_app  | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256 | <my_app_server_ip>
    51767 | postgres  | my_app  | f   |         |                        |      |
    ```
