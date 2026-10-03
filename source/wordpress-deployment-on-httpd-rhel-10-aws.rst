################################################
WordPress Deployment on HTTPD, RHEL 10, AWS
################################################

************
Introduction
************

The objective is to deploy WordPress in production on AWS using the classic LAMP stack based on RHEL 10, without containerization, with maximum emphasis on configuration security at the OS, web server, and PHP levels.

The site runs under Apache (HTTPD) with the MPM Event module, with PHP handled via PHP-FPM over a Unix socket; MariaDB is deployed locally on the same instance. Access to the server is restricted by IP through Security Groups, direct requests to the web server by IP address (bypassing the domain name) are blocked by a separate default host, and TLS certificates are issued and renewed automatically via Certbot. Additionally, security headers are configured, PHP execution in the uploads directory is disabled, permissions on wp-config.php are restricted, and separate SELinux contexts are set for configuration, logs, and site content.

Below is a step-by-step description of the entire cycle, from preparing the infrastructure and networking in AWS through installing and maintaining WordPress, to the final verification of DNS, HTTPS, redirects, and automatic certificate renewal.

---------



************
Architecture
************

.. graphviz::

   digraph wordpress_httpd{
       rankdir=TB;
       compound=true;
       labelloc="t";
       ranksep=1;
       fontname="monospace";
       node[fontname="monospace"];
       edge[fontname="monospace"];
       label="WordPress Deployment Architecture\n(HTTPD, RHEL 10, AWS, Let's Encrypt CA)";
       ca[label="Let's Encrypt CA"];
       user[label="Web User\n(HTTP/HTTPS)"];
       subgraph cluster_aws{
           label="Amazon Web Services";
           aws_route_53[label="AWS Route 53\n A record → AWS Elastic IP"];
           aws_eip[label="AWS Elastic IP"];
           aws_sg[label="AWS Security Group\n(allow 80/tcp, 443/tcp from 0.0.0.0/0)"];
           subgraph cluster_host{
               label="Host";
               certbot[label="Certbot"];
               httpd[label="HTTPD Apache"];
               php_fpm[label="PHP FPM"];
               db[label="Maria DB"];
           }
       }
       user->aws_route_53[label="DNS Lookup",style=dashed];
       user->aws_eip;
       aws_route_53->aws_eip;
       aws_eip->aws_sg;
       aws_sg->httpd;
       httpd->php_fpm->db;
       certbot->httpd[label="challenge file (webroot)",style=dashed,color=red];
       certbot->ca[label="ACME Certificate Request",style=dashed,color=green];
       ca->aws_eip->aws_sg->httpd[label="HTTP-01\nGet challenge",style=dashed,color=blue];
       ca->certbot[label="Issue certificate"];
       certbot->httpd[label="Install certificate"];
   }

---------



***********
Environment
***********

* Amazon Web Services
* Red Hat Enterprise Linux release 10.2 (Coughlan)
* kernel-6.12.0-211.26.1.el10_2.x86_64
* httpd-2.4.63-13.el10_2.1.x86_64
* mariadb-10.11.15-2.el10_1.x86_64
* php-8.3.31-1.el10_2.x86_64
* WordPress 7.0

---------



****************
AWS Preparation
****************

Security and Access Control Configuration
==========================================

1. Imported **Key Pair** ``default``, **Type** ``ed25519``
2. Created **Security Group** ``sg-<aws-sg-id> - ssh``
3. Added **Inbound rule** ``22/tcp``, **Source** ``My IP`` to **SG** ``sg-<aws-sg-id> - ssh``
4. Created **SG** ``sg-<aws-sg-id> - wordpress-web``
5. Added **Inbound rules** ``80/tcp``, ``443/tcp``, **Source** ``0.0.0.0/0`` to **SG** ``sg-<aws-sg-id> - wordpress-web``

---------

Deploying an EC2 Instance with Configured Access
=================================================

1. Created a **t3.small** instance ``wordpress-instance`` running RHEL 10
2. Applied **Key Pair** ``default``, **SG** ``sg-<aws-sg-id> - ssh``, **SG** ``sg-<aws-sg-id> - wordpress-web`` to instance ``wordpress-instance``

---------

Allocating and Associating an EIP
==================================

1. Created **Elastic IP address** ``<aws-eip>``
2. Associated **EIP** ``<aws-eip>`` with ``wordpress-instance``

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

============   ===========
Record name    www
Record type    A
Value          <aws-eip>
TTL            300
============   ===========

---------



*******************
System Preparation
*******************

Basic Configuration
====================

1. Connected to instance ``wordpress-instance``: ``$ ssh ec2-user@<aws-eip>``
2. Registered the system with RHSM: ``$ sudo subscription-manager register``
3. Updated the system: ``$ sudo dnf upgrade``
4. Rebooted the system after the ``kernel`` update: ``$ sudo systemctl reboot``
5. In the file :file:`/etc/rhsm/rhsm.conf`, changed the value ``manage_repos = 0`` to ``manage_repos = 1`` to allow ``subscription-manager`` to manage repositories
6. Enabled the **codeready-builder** repository: ``$ sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms``
7. Enabled the **EPEL repository for RHEL 10**: ``$ sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm``

---------

Package Installation
=====================

Installed the required packages.

.. code-block::

   $ sudo dnf install httpd mod_ssl mod_http2 mariadb-server php php-mysqlnd php-gd php-intl php-pecl-zip php-pecl-imagick certbot wget

   $ rpm -q httpd mod_ssl mod_http2 mariadb-server php php-mysqlnd php-gd php-intl php-pecl-zip php-pecl-imagick certbot wget
   httpd-2.4.63-13.el10_2.1.x86_64
   mod_ssl-2.4.63-13.el10_2.1.x86_64
   mod_http2-2.0.29-4.el10_2.1.x86_64
   mariadb-server-10.11.15-2.el10_1.x86_64
   php-8.3.31-1.el10_2.x86_64
   php-mysqlnd-8.3.31-1.el10_2.x86_64
   php-gd-8.3.31-1.el10_2.x86_64
   php-intl-8.3.31-1.el10_2.x86_64
   php-pecl-zip-1.22.3-5.el10.x86_64
   php-pecl-imagick-3.8.1-1.el10_2.x86_64
   certbot-4.2.0-1.el10_2.noarch
   wget-1.24.5-5.el10.x86_64

---------

Starting and Enabling the MariaDB and PHP-FPM Services
=========================================================

Enabled the ``mariadb`` and ``php-fpm`` services: ``$ sudo systemctl enable --now mariadb php-fpm``

---------



********************************
Basic HTTPD Apache Configuration
********************************

1. MPM Event is already enabled by default in the file :file:`/etc/httpd/conf.modules.d/00-mpm.conf`
2. Created the file :file:`/etc/httpd/conf.d/security.conf` to hide server information and disable the TRACE method:

.. code-block::

   # Hide the detailed software version in HTTP response headers
   ServerTokens Prod

   # Disable the server signature on error pages and directory listings
   ServerSignature Off

   # Disallow the insecure HTTP TRACE method (protection against Cross-Site Tracking)
   TraceEnable Off

3. Disabled the default SSL VirtualHost (from mod_ssl)

.. code-block::

   # Created a backup of the file
   $ sudo cp /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf.backup
   
   # Commented out the entire VirtualHost block in /etc/httpd/conf.d/ssl.conf
   sed -i '/<VirtualHost _default_:443>/,/<\/VirtualHost>/s/^/#/' /etc/httpd/conf.d/ssl.conf

---------



*********************
MariaDB Configuration
*********************

- Created **database** ``wordpress_prod``.
- Created **user** ``wordpress_prod_user``.
- Granted ``wordpress_prod_user`` full privileges (read, write, modify) on all tables in database ``wordpress_prod``.

.. code-block::

   $ sudo mysql -u root <<'SQL'
   CREATE DATABASE wordpress_prod
     CHARACTER SET utf8mb4
     COLLATE utf8mb4_unicode_ci;
   CREATE USER 'wordpress_prod_user'@'localhost' IDENTIFIED BY '<password>';
   GRANT ALL PRIVILEGES ON wordpress_prod.* TO 'wordpress_prod_user'@'localhost';
   FLUSH PRIVILEGES;
   SQL

---------



*********************
PHP-FPM Configuration
*********************

1. Created a dedicated pool :file:`/etc/php-fpm.d/example.com.conf`:

.. code-block::

   ; Pool name and system user/group for the process
   [example.com]
   user  = apache
   group = apache

   ; Socket configuration (connection to the web server)
   listen       = /run/php-fpm/example.com.sock
   listen.owner = apache
   listen.group = apache
   listen.mode  = 0660

   ; Process Manager
   pm                   = dynamic
   pm.max_children      = 15
   pm.start_servers     = 3
   pm.min_spare_servers = 2
   pm.max_spare_servers = 8
   pm.max_requests      = 500

   ; Pool-specific PHP directives and limits
   php_admin_value[error_log]            = /var/log/php-fpm/example.com-error.log
   php_admin_flag[log_errors]            = on
   php_admin_value[upload_max_filesize]  = 64M
   php_admin_value[post_max_size]        = 64M
   php_admin_value[memory_limit]         = 256M
   php_admin_value[max_execution_time]   = 300
   php_admin_value[session.save_path]    = /var/lib/php/session

2. Disabled the default ``www`` pool. ``$ sudo mv /etc/php-fpm.d/www.conf /etc/php-fpm.d/www.conf.disabled``
3. Applied the PHP-FPM settings. ``$ sudo systemctl restart php-fpm``

---------



*********************
OPcache Configuration
*********************

Added new values to the existing file :file:`/etc/php.d/10-opcache.ini`:

.. code-block::

   ; WordPress-optimized OPcache
   opcache.memory_consumption      = 128
   opcache.interned_strings_buffer = 16
   opcache.max_accelerated_files   = 10000
   opcache.revalidate_freq         = 2
   opcache.save_comments           = 1

---------



*********************
WordPress Preparation
*********************

1. Created the directories: ``$ sudo mkdir -p /var/www/example.com/{public_html,logs}``
2. Downloaded, extracted, and copied WordPress to the directory /var/www/example.com/public_html

.. code-block::

   $ cd /tmp
   $ wget https://wordpress.org/latest.tar.gz
   $ tar xzf latest.tar.gz
   $ sudo cp -a wordpress/. /var/www/example.com/public_html/
   $ rm -rf wordpress latest.tar.gz

3. Copied the file :file:`wp-config.php` one level above **DocumentRoot**: ``$ sudo cp /var/www/example.com/public_html/wp-config-sample.php /var/www/example.com/wp-config.php``
4. Changed the owner and group, and configured permissions on files and directories

.. code-block::

   $ sudo chown -R apache:apache /var/www/example.com/
   $ sudo find /var/www/example.com/public_html -type f -exec chmod 644 {} \;
   $ sudo find /var/www/example.com/public_html -type d -exec chmod 755 {} \;

5. Installed WP-CLI

.. code-block::

   $ sudo curl -sO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
   $ sudo chmod +x wp-cli.phar
   $ sudo mv wp-cli.phar /usr/local/bin/wp

6. Set the database parameters in the file :file:`wp-config.php`

.. code-block::

   $ sudo sed -i "s/database_name_here/wordpress_prod/" /var/www/example.com/wp-config.php
   $ sudo sed -i "s/username_here/wordpress_prod_user/" /var/www/example.com/wp-config.php
   $ sudo sed -i "s/password_here/<password>/" /var/www/example.com/wp-config.php

7. Generated security keys: ``$ sudo curl -s https://api.wordpress.org/secret-key/1.1/salt/``
8. Added the security keys to the file :file:`/var/www/example.com/wp-config.php`

.. code-block::

   define('AUTH_KEY',         <auth-key>);
   define('SECURE_AUTH_KEY',  <secure-auth-key>);
   define('LOGGED_IN_KEY',    <logged-in-key>);
   define('NONCE_KEY',        <nonce-key>);
   define('AUTH_SALT',        <auth-salt>);
   define('SECURE_AUTH_SALT', <secure-auth-salt>);
   define('LOGGED_IN_SALT',   <logged-in-salt>);
   define('NONCE_SALT',       <nonce-salt>);

9. Set the following configuration in the file :file:`/var/www/example.com/wp-config.php`:

.. code-block::

   // Site addresses (hardcoding the URL in code protects against failures and reduces DB load)
   define( 'WP_HOME',    'https://example.com' ); // Site home page
   define( 'WP_SITEURL', 'https://example.com' ); // WordPress files folder

   // Security and automatic updates
   define( 'FORCE_SSL_ADMIN',    true );    // Admin login only over secure HTTPS
   define( 'DISALLOW_FILE_EDIT', true );    // Disable the code editor in admin (protection against compromise)
   define( 'WP_AUTO_UPDATE_CORE', 'minor' ); // Auto-install only minor security updates

   // File write method
   define( 'FS_METHOD', 'direct' ); // Update plugins directly (without prompting for an FTP password)

   // Debugging (disabled for the production site)
   define( 'WP_DEBUG',         false ); // Disable debug mode
   define( 'WP_DEBUG_LOG',     false ); // Disable writing errors to a file on the server
   define( 'WP_DEBUG_DISPLAY', false ); // Disable displaying errors to site visitors

10. Changed ``$table_prefix`` from ``$table_prefix = 'wp_';`` to ``$table_prefix = 'example_';`` in the file :file:`/var/www/example.com/wp-config.php`
11. Set permissions on the file :file:`/var/www/example.com/wp-config.php`: ``$ sudo chmod 400 /var/www/example.com/wp-config.php``

---------



*********************
SELinux Configuration
*********************

SELinux was configured to allow unobstructed core, plugin, and theme updates directly through the admin panel without Site Health warnings, while still preserving baseline protection for the configuration file and logs.

.. code-block::

   # Allow write access to the directory by default
   $ sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/example\.com(/.*)?"

   # Exclude wp-config.php (read-only)
   $ sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/example\.com/wp-config\.php"

   # Exclude logs (set type httpd_log_t)
   $ sudo semanage fcontext -a -t httpd_log_t "/var/www/example\.com/logs(/.*)?"

   # Applied the changes
   $ sudo restorecon -Rv /var/www/example.com/

   # Allow network activity
   $ sudo setsebool -P httpd_can_network_connect 1

---------



************************************************************
HTTP Virtual Host Configuration (Before Configuring Certbot)
************************************************************

1. Created a temporary HTTP virtual host file :file:`/etc/httpd/conf.d/example.com.conf`:

.. code-block::

   <VirtualHost *:80>
       ServerName  example.com
       ServerAlias www.example.com
       DocumentRoot /var/www/example.com/public_html

       <FilesMatch \.php$>
           SetHandler "proxy:unix:/run/php-fpm/example.com.sock|fcgi://localhost"
       </FilesMatch>

       <Directory /var/www/example.com/public_html>
           Options       -Indexes
           AllowOverride All
           Require       all granted
       </Directory>

       ErrorLog  /var/www/example.com/logs/error.log
       CustomLog /var/www/example.com/logs/access.log combined
   </VirtualHost>

2. Verified the **httpd** configuration: ``$ sudo httpd -t``
3. Enabled and started the **httpd** service: ``$ sudo systemctl enable --now httpd``

---------



*********************
Certbot Configuration
*********************

1. Requested certificates for example.com and www.example.com

.. code-block::

   sudo certbot certonly \
     --webroot \
     --webroot-path /var/www/example.com/public_html \
     -d example.com \
     -d www.example.com \
     --non-interactive \
     --agree-tos \
     --email security@example.com

2. Started the **certbot-renew.timer** service: ``$ sudo systemctl start certbot-renew.timer``; the **certbot-renew.timer** timer is enabled by default (preset: enabled)
3. All simulated certificate renewals completed successfully: ``$ sudo certbot renew --dry-run``

---------



**********************************************
Configuring the Block on Direct IP Access
**********************************************

Created the file :file:`/etc/httpd/conf.d/00-default.conf` to block direct requests to the server by IP address instead of the domain name.

.. code-block::

   <VirtualHost *:80>
       ServerName _default_

       <Location />
           Require all denied
       </Location>
   </VirtualHost>

   <VirtualHost *:443>
       ServerName _default_
       SSLEngine on
       SSLCertificateFile    /etc/letsencrypt/live/example.com/fullchain.pem
       SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

       <Location />
           Require all denied
       </Location>
   </VirtualHost>

---------



**********************************
HTTPS Virtual Host Configuration
**********************************

1. After requesting the certificates, the contents of the temporary file :file:`/etc/httpd/conf.d/example.com.conf` were changed to:

.. code-block::

   ##########################################################################
   # HTTP (Port 80): Redirect to HTTPS with an exception for Let's Encrypt
   ##########################################################################

   <VirtualHost *:80>
       ServerName  example.com
       ServerAlias www.example.com

       # Explicit path for domain ownership verification (Certbot / ACME webroot).
       # Excluding it from the redirect ensures the challenge succeeds even if TLS is not yet working.
       Alias /.well-known/acme-challenge \
             /var/www/example.com/public_html/.well-known/acme-challenge
       <Directory /var/www/example.com/public_html/.well-known/acme-challenge>
           Require all granted
       </Directory>

       # Redirect all requests, except the ACME challenge, to the canonical HTTPS URL
       RewriteEngine On
       RewriteCond   %{REQUEST_URI} !^/.well-known/acme-challenge/
       RewriteRule   ^ https://example.com%{REQUEST_URI} [R=301,L]
   </VirtualHost>



   #########################################################################
   # HTTPS (Port 443): Redirect www.example.com → https://example.com
   #########################################################################

   # Note: duplicating the certificates here is necessary because a secure
   # TLS connection must be established successfully before the server can issue the redirect.
   <VirtualHost *:443>
       ServerName www.example.com

       SSLEngine on
       SSLCertificateFile    /etc/letsencrypt/live/example.com/fullchain.pem
       SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

       # Supported transfer protocols (HTTP/2 preferred)
       Protocols h2 http/1.1

       # Exact redirect that reliably preserves the request path without using mod_rewrite
       RedirectMatch permanent ^/(.*)$ https://example.com/$1
   </VirtualHost>



   ######################################################################
   # HTTPS (Port 443): Main canonical host example.com (WordPress)
   ######################################################################

   <VirtualHost *:443>
       ServerName   example.com
       DocumentRoot /var/www/example.com/public_html

       # Enable SSL and paths to the Let's Encrypt certificates
       SSLEngine on
       SSLCertificateFile    /etc/letsencrypt/live/example.com/fullchain.pem
       SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

       # Mozilla Intermediate security profile (supports TLS 1.2 and TLS 1.3)
       # The SSLCipherSuite directive is written on a single line for correct Apache parsing
       SSLProtocol         all -SSLv3 -TLSv1 -TLSv1.1
       SSLCipherSuite      ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
       SSLHonorCipherOrder off
       SSLSessionTickets   off

       # Supported transfer protocols (HTTP/2 preferred)
       Protocols h2 http/1.1

       # Security headers
       # HSTS: 300 seconds for testing purposes. Once the site's stable operation is confirmed,
       # it is recommended to change this to: "max-age=63072000; includeSubDomains; preload"
       Header always set Strict-Transport-Security  "max-age=300; includeSubDomains"
       Header always set X-Frame-Options            "SAMEORIGIN"
       Header always set X-Content-Type-Options     "nosniff"
       Header always set Referrer-Policy            "strict-origin-when-cross-origin"
       Header always set Permissions-Policy         "geolocation=(), microphone=(), camera=()"
       Header always unset X-Powered-By

       # Hand off PHP script processing to PHP-FPM via a Unix socket
       <FilesMatch \.php$>
           SetHandler "proxy:unix:/run/php-fpm/example.com.sock|fcgi://localhost"
       </FilesMatch>

       # Basic access settings for the site directory
       <Directory /var/www/example.com/public_html>
           Options       -Indexes
           AllowOverride All
           Require       all granted
       </Directory>

       # Security: disallow execution of PHP scripts in the WordPress uploads directory
       # (minimizes risk from uploaded vulnerable files or web shells)
       <DirectoryMatch "/var/www/example\.com/public_html/wp-content/uploads">
           <FilesMatch "\.php$">
               Require all denied
           </FilesMatch>
       </DirectoryMatch>

       # Compress transferred text content (mod_deflate module)
       AddOutputFilterByType DEFLATE \
         text/html text/plain text/xml text/css \
         application/javascript application/json application/xml+rss

       # Request and error logging
       ErrorLog  /var/www/example.com/logs/ssl-error.log
       CustomLog /var/www/example.com/logs/ssl-access.log combined
   </VirtualHost>

2. Applied the changes: ``$ sudo systemctl reload httpd``

---------



**********************
WordPress Installation
**********************

.. code-block::

   $ cd /var/www/example.com/public_html

   $ sudo -u apache /usr/local/bin/wp core install \
     --url="https://example.com" \
     --title="example" \
     --admin_user="<wp-admin-username>" \
     --admin_password="<password>" \
     --admin_email="webmaster@example.com" \
     --skip-email

Set the WordPress permalink structure: ``$ sudo -u apache /usr/local/bin/wp rewrite structure '/%postname%/' --hard``

---------



*********************
Configuring .htaccess
*********************

1. Created the file :file:`/var/www/example.com/public_html/.htaccess`:

.. code-block::

   # Block direct access to configuration and core information files
   <FilesMatch "^(wp-config\.php|readme\.html|license\.txt)$">
       Require all denied
   </FilesMatch>

   # Configure redirect rules (mod_rewrite module)
   <IfModule mod_rewrite.c>
       RewriteEngine On
       RewriteBase /

       # Disallow execution of PHP scripts in the uploads directory (protection against shell uploads)
       RewriteCond %{REQUEST_URI} ^/wp-content/uploads/.*\.php$ [NC]
       RewriteRule . - [F,L]

       # WordPress single entry point (route to index.php for non-existent files/folders)
       RewriteRule ^index\.php$ - [L]
       RewriteCond %{REQUEST_FILENAME} !-f
       RewriteCond %{REQUEST_FILENAME} !-d
       RewriteRule . /index.php [L]
   </IfModule>

2. Changed the owner of the file :file:`/var/www/example.com/public_html/.htaccess`: ``$ sudo chown apache:apache /var/www/example.com/public_html/.htaccess``
3. Set the file :file:`/var/www/example.com/public_html/.htaccess` to read-only

.. code-block::

   # Added a rule to the SELinux database (read-only)
   $ sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/example\.com/public_html/\.htaccess"

   # Apply the context to the actual file
   $ sudo restorecon -v /var/www/example.com/public_html/.htaccess

---------



**********************
WordPress Maintenance
**********************

Updating Core, Plugins, and Themes
=====================================

.. code-block::

   $ cd /var/www/example.com/public_html
   $ sudo -u apache /usr/local/bin/wp core update
   $ sudo -u apache /usr/local/bin/wp plugin update --all
   $ sudo -u apache /usr/local/bin/wp theme update --all

---------

Removing Unused Components (Themes and Plugins)
===================================================

.. code-block::

   $ cd /var/www/example.com/public_html
   $ sudo -u apache /usr/local/bin/wp theme list
   $ sudo -u apache /usr/local/bin/wp theme delete twentytwentyfour twentytwentythree

   # Removed inactive plugins.
   $ sudo -u apache bash -c '/usr/local/bin/wp plugin delete $(/usr/local/bin/wp plugin list --status=inactive --field=name)'

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

MariaDB Status
==================

.. code-block::

   $ systemctl status mariadb
   ● mariadb.service - MariaDB 10.11 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-06-22 12:26:39 UTC; 34min ago

---------

PHP-FPM Status
==================

.. code-block::

   $ systemctl status php-fpm
   ● php-fpm.service - The PHP FastCGI Process Manager
     Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-06-22 12:26:36 UTC; 35min ago

---------

MPM Event is Enabled
========================

.. code-block::

   # Check the active MPM
   httpd -V | grep -i "MPM"
   Server MPM:     event

   # Check the MPM configuration
   $ grep "mpm_" /etc/httpd/conf.modules.d/00-mpm.conf
   #LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
   #LoadModule mpm_worker_module modules/mod_mpm_worker.so
   LoadModule mpm_event_module modules/mod_mpm_event.so

---------

HTTPD Status
================

.. code-block::

   $ systemctl status httpd
   ● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Drop-In: /etc/systemd/system/httpd.service.d
            └─php-fpm.conf
     Active: active (running) since Tue 2026-06-23 11:42:06 UTC; 19s ago

---------

certbot-renew Status
========================

.. code-block::

   $ systemctl status certbot-renew.timer
   ● certbot-renew.timer - This is the timer to set the schedule for automated renewals
     Loaded: loaded (/usr/lib/systemd/system/certbot-renew.timer; enabled; preset: enabled)
     Active: active (waiting) since Tue 2026-06-23 11:47:32 UTC; 1min 53s ago

---------

certbot-renew Completed Successfully
========================================

.. code-block::

   $ sudo certbot renew --dry-run
   Saving debug log to /var/log/letsencrypt/letsencrypt.log

   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Processing /etc/letsencrypt/renewal/example.com.conf
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Account registered.
   Simulating renewal of an existing certificate for example.com and www.example.com

   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Congratulations, all simulated renewals succeeded:
     /etc/letsencrypt/live/example.com/fullchain.pem (success)
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

---------

Web Server Configuration Syntax is Correct
==============================================

.. code-block::

   $ sudo httpd -t
   Syntax OK

---------

No SELinux Denials or Security Errors
=========================================

.. code-block::

   $ sudo ausearch -m avc -ts recent | audit2why
   <no matches>
   Nothing to do

---------

Security Headers are Served Correctly
=========================================

.. code-block::

   $ curl -sI https://example.com | grep -iE "strict-transport|x-frame|x-content|referrer|permissions"
   strict-transport-security: max-age=300; includeSubDomains
   x-frame-options: SAMEORIGIN
   x-content-type-options: nosniff
   referrer-policy: strict-origin-when-cross-origin
   permissions-policy: geolocation=(), microphone=(), camera=()

---------

Redirect Chain
==================

.. code-block::

   # Configured a permanent redirect (301) from HTTP to secure HTTPS for the main domain
   $ curl -sI http://example.com       | grep -E "^HTTP|^Location"
   HTTP/1.1 301 Moved Permanently
   Location: https://example.com/

   # Configured a permanent redirect (301) from the HTTP www version to the HTTPS non-www version
   $ curl -sI http://www.example.com   | grep -E "^HTTP|^Location"
   HTTP/1.1 301 Moved Permanently
   Location: https://example.com/

   # The redirect chain for the HTTPS www version ends with a successful response (200 OK) over HTTP/2
   $ curl -sIL https://www.example.com | grep -E "^HTTP|^Location"
   HTTP/2 301
   HTTP/2 200

   # HTTP/2 protocol support for the tested domain is active and working correctly
   $ curl -sI --http2 https://example.com | head -1
   HTTP/2 200
