################################################
Ghost CMS Deployment on AWS, Debian, Docker
################################################

************
Introduction
************

The objective is to deploy Ghost CMS in production on AWS with full service isolation and a minimal attack surface.

The application and the database run in Docker containers and have no direct exposure to the internet: Ghost listens only on the loopback interface (127.0.0.1:2368), and MySQL is reachable only within the private Docker network. The single external entry point is NGINX, which terminates TLS and proxies requests to the container.

Below is a step-by-step description of the entire cycle, from preparing the infrastructure in AWS to the final verification of TLS, redirects, and automatic backups.

---------



************
Architecture
************

.. graphviz::

   digraph ghost{
      rankdir=TB;
      newrank=true;
      compound=true;
      labelloc="t";
      ranksep=1;
      fontname="monospace";
      label="GHOST CMS Deployment Architecture\n(Docker, Debian 13, AWS)";
      node[fontname="monospace"];
      edge[fontname="monospace"];
      user[label="Web User\nHTTP/HTTPS\nexample.com"];
      subgraph cluster_aws{
         label="Amazon Web Services";
         aws_eip[label="Elastic IP\n<aws-eip>"];
         aws_route_53[label="AWS Route 53\n A record → AWS Elastic IP"];
         aws_sg[label="Security Group\nsg-<aws-sg-id> - ghost-web\nInbound Rules - 80/tcp, 443 tcp\n Source 0.0.0.0/0"];
         aws_iam_role[label="IAM Role\nghost-ec2-s3-backup"];
         aws_s3_bucket[label="S3 Bucket\nghost-backup-id"];
         subgraph cluster_host{
            label="Host\nghost-instance";
            ufw[label="UFW\nAllow 22,80,443/tcp"]
            nginx[label="NGINX\nReverse Proxy\nTLS Termination"];
            subgraph cluster_docker_network{
               label="Network\nghost_net";
               ghost[label="GHOST\n:2368"];
               mysql[label="MySQL\n:3306"];
            }
         }
      }
      user->aws_route_53[label="DNS Lookup",style=dashed];
      aws_route_53->aws_eip;
      user->aws_eip;
      aws_eip->aws_sg;
      aws_sg->ufw[lhead=cluster_host];
      ufw->nginx;
      nginx->ghost[label="127.0.0.1:2368",lhead=cluster_docker_network];
      ghost->mysql;
      aws_iam_role->aws_s3_bucket[label="S3 API"];
      mysql->aws_iam_role[label="Instance Profile",ltail=cluster_host];
      {rank="same"; ghost; mysql;}
      {rank="same"; ufw; nginx;}
      {rank="same"; aws_iam_role; aws_s3_bucket;}
      {rank="same"; aws_route_53; aws_eip;}
   }

---------



***********
Environment
***********

* Amazon Web Services
* Debian GNU/Linux 13 (trixie)
* 6.12.74+deb13+1-cloud-amd64
* Docker Engine - Community 29.6.1
* Docker Compose v5.2.0

---------



****************
Container Images
****************

* ghost:6.49.0-bookworm
* mysql:8.4.10

---------



****************
AWS Preparation
****************

Security and Access Control Configuration
==========================================

1. Imported **Key Pair** ``default``, **Type** ``ed25519``
2. Created **Security Group** ``sg-<aws-sg-id> - ssh``
3. Added **Inbound rules** ``22/tcp``, **Source** ``My IP`` to **SG** ``sg-<aws-sg-id> - ssh``
4. Created **SG** ``sg-<aws-sg-id> - ghost-web``
5. Added **Inbound rules** ``80/tcp``, ``443/tcp``, **Source** ``0.0.0.0/0`` to **SG** ``sg-<aws-sg-id> - ghost-web``

---------

Deploying an EC2 Instance with Configured Access
=================================================

1. Created a **t3.small** instance ``ghost-instance``
2. Applied **Key Pair** ``default``, **SG** ``sg-<aws-sg-id> - ssh``, **SG** ``sg-<aws-sg-id> - ghost-web`` to instance ``ghost-instance``

---------

Allocating and Associating an EIP
==================================

1. Created **Elastic IP address** ``<aws-eip>``
2. Associated **EIP** ``<aws-eip>`` with ``ghost-instance``

---------

DNS Configuration in Route 53: Creating a Hosted Zone and A Records
=====================================================================

1. Created **Public hosted zone** ``example.com``
2. Configured **Route 53** nameservers, ``ns-1.awsdns.com.``, ``ns-22.awsdns.co.uk.``, ``ns-333.awsdns.org.``, ``ns-444.awsdns.net.`` for domain ``example.com`` in the registrar's control panel
3. Created an ``A`` record ``example.com`` in **Public hosted zone** ``example.com``

===========   ===========
Record name   example.com
Record type   A
Value         <aws-eip>
TTL           300
===========   ===========

4. Created an ``A`` record ``www``

===========   =========
Record name   www
Record type   A
Value         <aws-eip>
TTL           300
===========   =========

---------

Configuring an S3 Bucket for Backups
=====================================

1. Created **S3 Bucket** ``ghost-backup-id``
2. Created **Lifecycle rule** ``delete-after-30-days`` for **S3 Bucket** ``ghost-backup-id`` with the following parameters:

==================================   ==================================
Rule scope                           Apply to all objects in the bucket
Lifecycle rule actions               Expire current versions of objects
Expire current versions of objects   Days after object creation: 30
Block all public access              Enabled
Default encryption                   SSE-S3
==================================   ==================================

---------

Configuring IAM for Backups to S3
===================================

1. Created **IAM Role** ``ghost-ec2-s3-backup`` with the following parameters:

===================   ===========
Trusted entity type   AWS service
Use case              EC2
===================   ===========

2. Configured **Trust Policy** for **IAM Role** ``ghost-ec2-s3-backup``:

.. code-block::

   {
      "Version": "2012-10-17",
      "Statement": [
         {
            "Effect": "Allow",
            "Principal": {
               "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
         }
      ]
   }

3. Created **Permission Policy** ``ghost-backup-s3-policy`` in **IAM Role** ``ghost-ec2-s3-backup``:

.. code-block::

   {
      "Version": "2012-10-17",
      "Statement": [
         {
            "Sid": "ListBucketAccess",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::ghost-backup-id"
         },
         {
            "Sid": "ObjectAccess",
            "Effect": "Allow",
            "Action": [
               "s3:PutObject",
               "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::ghost-backup-id/*"
         }
      ]
   }

4. Attached **IAM Role** ``ghost-ec2-s3-backup`` to instance ``ghost-instance`` via **Modify IAM role**

---------



***********************
Initial System Setup
***********************

Updating the System and Installing Required Packages
=======================================================

1. Updated the system: ``$ sudo apt update && sudo apt upgrade``
2. Installed packages ``ufw``, ``gnupg``, ``ca-certificates``, ``skopeo``, ``nginx``, ``certbot``, ``ssl-cert``

.. code-block::

   $ sudo apt install ufw gnupg ca-certificates skopeo nginx certbot

   $ dpkg -l ufw gnupg ca-certificates skopeo nginx certbot ssl-cert
   Desired=Unknown/Install/Remove/Purge/Hold
   | Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
   |/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
   ||/ Name            Version          Architecture
   +++-===============-================-============
   ii  ca-certificates 20250419         all
   ii  certbot         4.0.0-2+deb13u1  all
   ii  gnupg           2.4.7-21+deb13u1 all
   ii  nginx           1.26.3-3+deb13u7 amd64
   ii  skopeo          1.18.0+ds1-1+b5  amd64
   ii  ssl-cert        1.1.3            all
   ii  ufw             0.36.2-9         all

---------

Configuring UFW — Host-Level Firewall
=========================================

At the host level, all incoming connections are blocked by default, and all outgoing connections are allowed by default. Three ports, ``22/tcp``, ``80/tcp``, ``443/tcp``, are opened for both IPv4 and IPv6.

.. code-block::

   $ sudo ufw allow 22/tcp
   $ sudo ufw allow 80/tcp
   $ sudo ufw allow 443/tcp
   $ sudo ufw default deny incoming
   $ sudo ufw default allow outgoing
   $ sudo ufw enable

.. note::

   **ufw** allows ``22/tcp`` with no IP restriction, unlike **AWS SG**. Source restriction is enforced only at the **SG** level — since **SG** is managed through the **AWS Console** independently of the server, access can be restored after a dynamic IP change without risking loss of the **SSH** connection.

---------

Installing Docker CE
=====================

Installed packages:

* ``docker-ce``
* ``docker-ce-cli``
* ``containerd.io``
* ``docker-buildx-plugin``
* ``docker-compose-plugin``

.. code-block::

   $ sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)

   $ sudo install -m 0755 -d /etc/apt/keyrings
   $ sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
   $ sudo chmod a+r /etc/apt/keyrings/docker.asc

   $ sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
   Types: deb
   URIs: https://download.docker.com/linux/debian
   Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
   Components: stable
   Architectures: $(dpkg --print-architecture)
   Signed-By: /etc/apt/keyrings/docker.asc
   EOF

   $ sudo apt update

   $ sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

   $ dpkg -l docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   Desired=Unknown/Install/Remove/Purge/Hold
   | Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
   |/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
   ||/ Name                  Version                     Architecture
   +++-=====================-===========================-============
   ii  containerd.io         2.2.5-1~debian.13~trixie    amd64
   ii  docker-buildx-plugin  0.35.0-1~debian.13~trixie   amd64
   ii  docker-ce             5:29.6.1-1~debian.13~trixie amd64
   ii  docker-ce-cli         5:29.6.1-1~debian.13~trixie amd64
   ii  docker-compose-plugin 5.2.0-1~debian.13~trixie    amd64

---------

Docker Log Rotation
=====================

Created the file :file:`/etc/docker/daemon.json` to prevent container logs from filling the disk over time.

.. code-block::

   sudo tee /etc/docker/daemon.json <<'EOF'
   {
     "log-driver": "json-file",
     "log-opts": {
         "max-size": "10m",
         "max-file": "3"
      }
   }
   EOF

---------



****************************************************
Deploying Ghost + MySQL via Docker Compose
****************************************************

1. Created the directory structure

.. code-block::

   $ sudo mkdir -p /opt/ghost/scripts
   $ sudo chown -R admin:admin /opt/ghost
   $ cd /opt/ghost

2. Created the file :file:`/opt/ghost/.env` with the following contents:

.. code-block::

   $ sudo cat > /opt/ghost/.env <<'EOF'
   DOMAIN=example.com

   # Pinned container image versions
   GHOST_IMAGE=ghost:6.49.0-bookworm
   MYSQL_IMAGE=mysql:8.4.10

   # MySQL credentials
   MYSQL_ROOT_PASSWORD='<password>'
   MYSQL_DATABASE=ghost_production
   MYSQL_USER=ghost_production_user
   MYSQL_PASSWORD='<password>'

   # S3 Bucket name for backups
   S3_BACKUP_BUCKET=ghost-backup-id
   EOF

3. Changed permissions on the file :file:`/opt/ghost/.env`: ``$ chmod 600 /opt/ghost/.env``

4. Created the Docker Compose file :file:`/opt/ghost/docker-compose.yml`

.. code-block::

   $ sudo cat > /opt/ghost/docker-compose.yml <<'EOF'
   name: ghost

   services:

      db:
         image: ${MYSQL_IMAGE}
         container_name: ghost_db
         restart: always
         environment:
            MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
            MYSQL_DATABASE: ${MYSQL_DATABASE}
            MYSQL_USER: ${MYSQL_USER}
            MYSQL_PASSWORD: ${MYSQL_PASSWORD}
         volumes:
            - ghost_mysql_data:/var/lib/mysql
         networks:
            - ghost_net
         healthcheck:
            test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
            interval: 10s
            timeout: 5s
            retries: 5
            start_period: 30s

      ghost:
         image: ${GHOST_IMAGE}
         container_name: ghost_app
         restart: always
         depends_on:
            db:
               condition: service_healthy
         environment:
            NODE_ENV: production
            url: https://${DOMAIN}
            database__client: mysql
            database__connection__host: db
            database__connection__user: ${MYSQL_USER}
            database__connection__password: ${MYSQL_PASSWORD}
            database__connection__database: ${MYSQL_DATABASE}
         ports:
            - "127.0.0.1:2368:2368"
         volumes:
            - ghost_app_data:/var/lib/ghost/content
         networks:
            - ghost_net

   networks:
      ghost_net:
         driver: bridge

   volumes:
      ghost_mysql_data:
         driver: local
         name: ghost_mysql_data
      ghost_app_data:
         driver: local
         name: ghost_app_data
   EOF

5. Started the services

.. code-block::

   $ cd /opt/ghost
   $ sudo docker compose up -d
   $ sudo docker compose ps

---------



*****************************************
Configuring Nginx HTTP (Before Certbot)
*****************************************

1. Created the file :file:`/etc/nginx/sites-available/00-catch-all` to handle requests that don't match any ``server_name``

.. code-block::

   $ sudo tee /etc/nginx/sites-available/00-catch-all <<'EOF'
   #############################################################
   # Catch-all: block access by IP or unknown domain
   #############################################################
   server {
      listen 80 default_server;
      listen [::]:80 default_server;
      listen 443 ssl default_server;
      listen [::]:443 ssl default_server;
      server_name _;

      # Self-signed placeholder certificate, not Let's Encrypt
      ssl_certificate     /etc/ssl/certs/ssl-cert-snakeoil.pem;
      ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;

      return 444;
   }
   EOF

2. Created the directory :file:`/var/www/certbot` for Certbot Let's Encrypt certificates: ``sudo mkdir -p /var/www/certbot``

3. Created the file :file:`/etc/nginx/sites-available/01-example`

.. code-block::

   $ sudo tee /etc/nginx/sites-available/01-example <<'EOF'
   server {
      listen 80;
      listen [::]:80;
      server_name example.com www.example.com;

      # ACME challenge
      location /.well-known/acme-challenge/ {
         root /var/www/certbot;
      }

      location / {
         return 200 'Setup in progress';
         add_header Content-Type text/plain;
      }
   }

4. Enabled the configuration files ``00-catch-all``, ``01-example``, and disabled Nginx's default configuration file ``default``

.. code-block::

   $ sudo ln -s /etc/nginx/sites-available/00-catch-all /etc/nginx/sites-enabled/00-catch-all
   $ sudo ln -s /etc/nginx/sites-available/01-example /etc/nginx/sites-enabled/01-example
   $ sudo rm /etc/nginx/sites-enabled/default

5. Reloaded the Nginx configuration: ``$ sudo systemctl reload nginx``

---------



***************************************
Configuring Nginx HTTPS (with Certbot)
***************************************

1. Obtained Certbot certificates

.. code-block::

   $ sudo certbot certonly \
   --webroot \
   --webroot-path /var/www/certbot \
   -d example.com \
   -d www.example.com \
   --email security@example.com \
   --agree-tos \
   --no-eff-email

2. The temporary configuration :file:`/etc/nginx/sites-available/01-example` was replaced with the contents below, containing three server blocks:

.. code-block::

   ##########################################
   # HTTP: ACME challenge + redirect to HTTPS
   ##########################################
   server {
      listen 80;
      listen [::]:80;
      server_name example.com www.example.com;

      location /.well-known/acme-challenge/ {
         root /var/www/certbot;
      }

      location / {
         return 301 https://$host$request_uri;
      }
   }

   ####################################################
   # HTTPS: redirect www → non-www (canonical domain)
   ####################################################
   server {
      listen 443 ssl;
      listen [::]:443 ssl;
      http2 on;
      server_name www.example.com;

      ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

      ssl_protocols             TLSv1.2 TLSv1.3;
      ssl_prefer_server_ciphers off;
      ssl_session_cache         shared:SSL:10m;
      ssl_session_timeout       1d;
      ssl_session_tickets       off;

      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header X-Frame-Options           "SAMEORIGIN"                           always;
      add_header X-Content-Type-Options    "nosniff"                              always;
      add_header Referrer-Policy           "no-referrer-when-downgrade"           always;
      add_header X-XSS-Protection          "1; mode=block"                        always;

      return 301 https://example.com$request_uri;
   }

   ######################################
   # HTTPS: production (canonical domain)
   ######################################
   server {
      listen 443 ssl;
      listen [::]:443 ssl;
      http2 on;
      server_name example.com;

      ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

      ssl_protocols             TLSv1.2 TLSv1.3;
      ssl_prefer_server_ciphers off;
      ssl_session_cache         shared:SSL:10m;
      ssl_session_timeout       1d;
      ssl_session_tickets       off;

      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header X-Frame-Options           "SAMEORIGIN"                           always;
      add_header X-Content-Type-Options    "nosniff"                              always;
      add_header Referrer-Policy           "no-referrer-when-downgrade"           always;
      add_header X-XSS-Protection          "1; mode=block"                        always;

      location / {
         proxy_pass         http://127.0.0.1:2368;
         proxy_set_header   Host              $host;
         proxy_set_header   X-Real-IP         $remote_addr;
         proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
         proxy_set_header   X-Forwarded-Proto $scheme;
         proxy_buffering    off;
         proxy_read_timeout 90s;
      }
   }

3. Reloaded the Nginx configuration: ``$ sudo systemctl reload nginx``

4. Configured access to Ghost via the browser at ``https://example.com/ghost``

* username ``example@example.com``
* password ``<password>``

5. Created the file :file:`/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh` to reload the Nginx configuration after every successful Certbot certificate renewal.

.. code-block::

   $ sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh <<'EOF'
   #!/bin/bash
   systemctl reload nginx
   EOF

   $ sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh

---------



****************************
Configuring Backups
****************************

1. Created the backup script :file:`/opt/ghost/scripts/backup.sh`

.. code-block::

   #!/bin/bash
   set -euo pipefail

   ##############
   # Configuration
   ##############
   ENV_FILE="/opt/ghost/.env"
   TIMESTAMP=$(date -u +%Y%m%d_%H%M%S)
   STAGING="/tmp/ghost_backup_${TIMESTAMP}"
   LOG="/var/log/ghost_backup.log"

   #############
   # Logging
   ############# 
   log() { echo "[$(date -u '+%Y-%m-%d %H:%M:%S UTC')] $*" | tee -a "$LOG"; }
   die() { log "ERROR: $*"; exit 1; }

   ################################
   # Load environment variables
   ################################
   source "$ENV_FILE"

   log "=== Backup started: ${TIMESTAMP} ==="
   mkdir -p "$STAGING"

   ############
   # MySQL dump
   ############
   MYSQL_FILE="${STAGING}/mysql_${TIMESTAMP}.sql.gz"

   log "Creating MySQL dump..."
   docker exec -e MYSQL_PWD="${MYSQL_ROOT_PASSWORD}" ghost_db mysqldump \
      -u root \
      --single-transaction \
      --routines \
      --triggers \
      "${MYSQL_DATABASE}" \
      | gzip > "$MYSQL_FILE"

   [[ -s "$MYSQL_FILE" ]] || die "MySQL dump is empty"
   gzip -t "$MYSQL_FILE"   || die "MySQL dump archive is corrupted"
   log "MySQL dump OK: $(du -sh "$MYSQL_FILE" | cut -f1)"

   ############################
   # Ghost content (Docker volume)
   ############################
   GHOST_FILE="${STAGING}/ghost_content_${TIMESTAMP}.tar.gz"

   # Resolve the volume path dynamically — not tied to where Docker data lives
   GHOST_VOLUME_PATH=$(docker volume inspect ghost_app_data --format '{{ .Mountpoint }}')

   log "Archiving Ghost content from ${GHOST_VOLUME_PATH}..."
   tar -czf "$GHOST_FILE" -C "$GHOST_VOLUME_PATH" .

   [[ -s "$GHOST_FILE" ]] || die "Ghost content archive is empty"
   tar -tzf "$GHOST_FILE" > /dev/null || die "Ghost content archive is corrupted"
   log "Ghost content OK: $(du -sh "$GHOST_FILE" | cut -f1)"

   ################
   # Upload to S3
   ################
   S3_PATH="s3://${S3_BACKUP_BUCKET}/${TIMESTAMP}/"

   log "Uploading to S3: ${S3_PATH}"
   aws s3 cp "$STAGING/" "$S3_PATH" --recursive --quiet

   # Verification — expect both files on S3
   REMOTE_COUNT=$(aws s3 ls "$S3_PATH" | wc -l)
   [[ "$REMOTE_COUNT" -ge 2 ]] || die "S3 verification failed: expected 2 files, found ${REMOTE_COUNT}"

   log "S3 verification OK: ${REMOTE_COUNT} files at ${S3_PATH}"

   ##########################
   # Clean up temporary files
   ##########################
   rm -rf "$STAGING"
   log "=== Backup completed successfully ==="

   $ sudo chmod 700 /opt/ghost/scripts/backup.sh

2. Created the ``systemd`` service :file:`/etc/systemd/system/ghost-backup.service` and the ``systemd`` timer :file:`/etc/systemd/system/ghost-backup.timer` to automate backups.

.. code-block::

   $ sudo tee /etc/systemd/system/ghost-backup.service <<'EOF'
   [Unit]
   Description=Ghost Daily Backup
   After=docker.service network-online.target
   Requires=docker.service

   [Service]
   Type=oneshot
   ExecStart=/opt/ghost/scripts/backup.sh
   User=root
   StandardOutput=journal
   StandardError=journal
   SyslogIdentifier=ghost-backup
   EOF

.. code-block::

   $ sudo tee /etc/systemd/system/ghost-backup.timer <<'EOF'
   [Unit]
   Description=Ghost Daily Backup Timer

   # The server's initial time zone is UTC, +0000
   [Timer]
   OnCalendar=*-*-* 03:00:00 UTC
   Persistent=true

   [Install]
   WantedBy=timers.target
   EOF

3. Started and enabled the timer :file:`/etc/systemd/system/ghost-backup.timer`.

.. code-block::

   $ sudo systemctl daemon-reload
   $ sudo systemctl enable --now ghost-backup.timer

4. Created the file :file:`/etc/logrotate.d/ghost-backup` for rotating the backup log file.

.. code-block::

   $ sudo tee /etc/logrotate.d/ghost-backup <<'EOF'
   /var/log/ghost_backup.log {
      weekly
      rotate 4
      compress
      missingok
      notifempty
   }
   EOF

---------



************
Verification
************

Root Domain Verification
===========================

.. code-block::

   # Verify the root domain and www via Google DNS
   $ dig example.com @8.8.8.8 +short
   <aws-eip>

   $ dig www.example.com @8.8.8.8 +short
   <aws-eip>

   # Verify via Cloudflare DNS
   $ dig example.com @1.1.1.1 +short
   <aws-eip>

   $ dig www.example.com @1.1.1.1 +short
   <aws-eip>

---------

Verifying the IAM Role Attachment to the Instance
=====================================================

.. code-block::

   $ TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

   $ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
   ghost-ec2-s3-backup

---------

ufw Status
============

.. code-block::

   $ sudo ufw status verbose
   Status: active
   Logging: on (low)
   Default: deny (incoming), allow (outgoing), disabled (routed)
   New profiles: skip

   To                         Action      From
   --                         ------      ----
   22/tcp                     ALLOW IN    Anywhere
   80/tcp                     ALLOW IN    Anywhere
   443/tcp                    ALLOW IN    Anywhere
   22/tcp (v6)                ALLOW IN    Anywhere (v6)
   80/tcp (v6)                ALLOW IN    Anywhere (v6)
   443/tcp (v6)                ALLOW IN    Anywhere (v6)

---------

Docker Status
================

.. code-block::

   $ systemctl status docker
   ● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-07-01 11:54:14 UTC; 2min 9s ago

---------

Nginx Status
==============

.. code-block::

   $ systemctl status nginx
   ● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-07-02 09:57:32 UTC; 1 day 21h ago

---------

``certbot.timer`` Status
============================

.. code-block::

   $ sudo systemctl status certbot.timer
   ● certbot.timer - Run certbot twice daily
     Loaded: loaded (/usr/lib/systemd/system/certbot.timer; enabled; preset: enabled)
     Active: active (waiting) since Thu 2026-07-02 09:58:03 UTC; 1 day 21h ago

---------

The Certbot Hook is Correct
==============================

Certbot sees the hook at the correct path and recognizes it as a deploy hook, but deliberately skips running it because this is a dry run.

.. code-block::

   $ sudo cat /var/log/letsencrypt/letsencrypt.log | grep -i hook
   2026-07-04 07:25:46,265:INFO:certbot._internal.hooks:Dry run: skipping deploy hook command: /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh

---------

Backup Timer Status
=======================

.. code-block::

   $ sudo systemctl status ghost-backup.timer
   ● ghost-backup.timer - Ghost Daily Backup Timer
     Loaded: loaded (/etc/systemd/system/ghost-backup.timer; enabled; preset: enabled)
     Active: active (waiting) since Sat 2026-07-04 09:26:42 UTC; 23s ago

---------

Containers are Running
==========================

.. code-block::

   $ sudo docker compose -f /opt/ghost/docker-compose.yml ps
   NAME        IMAGE                   COMMAND                  SERVICE   CREATED      STATUS                PORTS
   ghost_app   ghost:6.49.0-bookworm   "docker-entrypoint.s…"   ghost     2 days ago   Up 2 days             127.0.0.1:2368->2368/tcp
   ghost_db    mysql:8.4.10            "docker-entrypoint.s…"   db        2 days ago   Up 2 days (healthy)   3306/tcp, 33060/tcp

---------

Ghost Responds over HTTPS
============================

.. code-block::

   $ curl -s -o /dev/null -w "HTTP %{http_code}\n" https://example.com
   HTTP 200

---------

The TLS Certificate is Valid
================================

.. code-block::

   $ echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
   notBefore=Jul  2 10:25:26 2026 GMT
   notAfter=Sep 30 10:25:25 2026 GMT

---------

The HTTP to HTTPS Redirect Works
====================================

.. code-block::

   $ curl -s -o /dev/null -w "Redirect: %{redirect_url}\n" http://example.com
   Redirect: https://example.com/

---------

Ghost Admin is Accessible
=============================

.. code-block::

   $ curl -s -o /dev/null -w "Admin HTTP %{http_code}\n" https://example.com/ghost
   Admin HTTP 301

   $ curl -sL -o /dev/null -w "Admin HTTP %{http_code}\n" https://example.com/ghost
   Admin HTTP 200

---------

The Backup Job Works
========================

.. code-block::

   $ sudo systemctl start ghost-backup.service
   $ sudo journalctl -u ghost-backup.service -f
   Jul 04 10:17:57 ip-172-31-2-94 ghost-backup[351740]: [2026-07-04 10:17:57 UTC] Creating MySQL dump...
   Jul 04 10:17:57 ip-172-31-2-94 ghost-backup[351773]: [2026-07-04 10:17:57 UTC] MySQL dump OK: 36K
   Jul 04 10:17:57 ip-172-31-2-94 ghost-backup[351783]: [2026-07-04 10:17:57 UTC] Archiving Ghost content from /var/lib/docker/volumes/ghost_app_data/_data...
   Jul 04 10:17:58 ip-172-31-2-94 ghost-backup[351794]: [2026-07-04 10:17:58 UTC] Ghost content OK: 816K
   Jul 04 10:17:58 ip-172-31-2-94 ghost-backup[351797]: [2026-07-04 10:17:58 UTC] Uploading to S3: s3://ghost-backup-id/20260704_101757/
   Jul 04 10:18:00 ip-172-31-2-94 ghost-backup[351843]: [2026-07-04 10:18:00 UTC] S3 verification OK: 2 files at s3://ghost-backup-id/20260704_101757/
   Jul 04 10:18:00 ip-172-31-2-94 ghost-backup[351847]: [2026-07-04 10:18:00 UTC] === Backup completed successfully ===
   Jul 04 10:18:00 ip-172-31-2-94 systemd[1]: ghost-backup.service: Deactivated successfully.
   Jul 04 10:18:00 ip-172-31-2-94 systemd[1]: Finished ghost-backup.service - Ghost Daily Backup.
   Jul 04 10:18:00 ip-172-31-2-94 systemd[1]: ghost-backup.service: Consumed 2.056s CPU time, 77.9M memory peak.
