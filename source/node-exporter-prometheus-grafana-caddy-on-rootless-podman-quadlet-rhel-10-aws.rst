###################################################################################
Node Exporter, Prometheus, Grafana, Caddy on Rootless Podman Quadlet, RHEL 10, AWS
###################################################################################

************
Introduction
************

The objective is to deploy a monitoring stack (Node Exporter, Prometheus, Grafana) in production on AWS with automatic TLS and minimal privileges at the OS level.

All services run in rootless Podman containers under the unprivileged user ``monitoring`` and are managed via Quadlet as regular systemd units. Node Exporter runs on the host network and reads host metrics directly, while Prometheus and Grafana are isolated on an internal ``bridge`` network and have no ports of their own exposed externally. The only external entry point is Caddy, which listens on ports ``80`` and ``443``, automatically issues and renews the TLS certificate, and proxies requests to Grafana.

Below is a step-by-step description of the entire cycle, from preparing the AWS infrastructure and configuring the system for rootless containers, through creating the Quadlet unit files, to the final verification after the instance is rebooted.

---------



************
Architecture
************

.. graphviz::

   digraph monitoring{
      rankdir=LR;
      compound=true;
      labelloc="t";
      ranksep=1;
      fontname="monospace";
      node[fontname="monospace"];
      edge[fontname="monospace"];
      label="Monitoring Stack Deployment Architecture\n(Node Exporter, Prometheus, Grafana, Caddy on rootless Podman Quadlet, RHEL 10, AWS)";
      internet[label="Internet"];
      subgraph cluster_aws{
         label="Amazon Web Services";
         aws_route_53[label="AWS Route 53"];
         aws_eip[label="AWS Elastic IP"];
         aws_sg[label="AWS Security Group"];
         subgraph cluster_host{
            label="RHEL 10 Host";
            subgraph cluster_quadlet_unit_files{
               label="Quadlet unit files\n(generate systemd units)\n~/.config/containers/systemd/";
               unit_file_network_monitoring[label="monitoring.network\n(creates internal monitoring network)"];
               unit_file_container_node_exporter[label="node-exporter.container\n(describes the Node Exporter container)"];
               unit_file_container_prometheus[label="prometheus.container\n(describes the Prometheus container)"];
               unit_file_container_grafana[label="grafana.container\n(describes the Grafana container)"];
               unit_file_container_caddy[label="caddy.container\n(describes the Caddy container)"];
               unit_file_volume_prometheus_data[label="prometheus-data.volume\n(Prometheus persistent volume)"];
               unit_file_volume_grafana_data[label="grafana-data.volume\n(Grafana persistent volume)"];
               unit_file_volume_caddy_data[label="caddy-data.volume\n(Caddy data persistent volume)"];
               unit_file_volume_caddy_config[label="caddy-config.volume\n(Caddy config persistent volume)"];
            }
            subgraph cluster_config_files{
               label="~/config/\n(bind-mount)";
               config_file_prometheus[label="prometheus/prometheus.yml\n(Prometheus configuration)"];
               config_file_grafana[label="grafana/grafana.env\n(Grafana configuration)"];
               config_file_grafana_data_source[label="grafana/provisioning/datasources/prometheus.yml\n(Grafana datasource)"];
               config_file_caddy[label="caddy/Caddyfile\n(Caddy configuration)"];
            }
            subgraph cluster_persistent_volumes{
               label="Persistent Volumes";
               persistent_volume_prometheus_data[label="Volume\nprometheus-data:/prometheus"];
               persistent_volume_grafana_data[label="Volume\ngrafana-data:/var/lib/grafana"];
               persistent_volume_caddy_data[label="Volume\ncaddy-data:/data"];
               persistent_volume_caddy_config[label="Volume\ncaddy-config:/config"];
            }
            subgraph cluster_network_host{
               label="Network host";
               container_node_exporter[label="Container Node Exporter\n9100/tcp"];
            }
            subgraph cluster_network_monitoring{
               label="Network monitoring";
               container_prometheus[label="Container Prometheus\n9099/tcp"];
               container_grafana[label="Container Grafana\n3000/tcp"];
               container_caddy[label="Container Caddy Web Server\n80/tcp, 443/tcp, 443/udp"];
            }
         }
      }
      // Global
      internet->aws_route_53[style=dotted];
      aws_route_53->aws_eip[style="dashed"];
      internet->aws_eip->aws_sg;
      aws_sg->container_caddy[lhead=cluster_host];
      unit_file_network_monitoring->container_caddy[lhead=cluster_network_monitoring];
      // container -> container
      container_caddy->container_grafana->container_prometheus;
      container_prometheus->container_node_exporter[label="host.containers.internal:9100",lhead=cluster_network_host,ltail=cluster_network_monitoring];
      // config_file -> unit_file_container
      config_file_prometheus->unit_file_container_prometheus[style="dashed"];
      config_file_grafana->unit_file_container_grafana[style="dashed"];
      config_file_grafana_data_source->unit_file_container_grafana[style="dashed"];
      config_file_caddy->unit_file_container_caddy[style="dashed"];
      // unit_file_volume -> persistent_volume
      unit_file_volume_prometheus_data->persistent_volume_prometheus_data;
      unit_file_volume_grafana_data->persistent_volume_grafana_data;
      unit_file_volume_caddy_data->persistent_volume_caddy_data;
      unit_file_volume_caddy_config->persistent_volume_caddy_config;
      // persistent_volumes -> container
      persistent_volume_prometheus_data->container_prometheus;
      persistent_volume_grafana_data->container_grafana;
      persistent_volume_caddy_data->container_caddy;
      persistent_volume_caddy_config->container_caddy;
      // unit_file_container -> container
      unit_file_container_node_exporter->container_node_exporter;
      unit_file_container_prometheus->container_prometheus;
      unit_file_container_grafana->container_grafana;
      unit_file_container_caddy->container_caddy;
      // unit_file_volume -> unit_file_container
      unit_file_volume_prometheus_data->unit_file_container_prometheus[style=dotted];
      unit_file_volume_grafana_data->unit_file_container_grafana[style=dotted];
      unit_file_volume_caddy_data->unit_file_container_caddy[style=dotted];
      unit_file_volume_caddy_config->unit_file_container_caddy[style=dotted];
   }

---------



***********
Environment
***********

* Amazon Web Services
* Red Hat Enterprise Linux release 10.2 (Coughlan)
* kernel-6.12.0-211.20.1.el10_2.x86_64
* podman-5.8.2-1.el10_2.x86_64

---------



****************
Container Images
****************

* quay.io/prometheus/node-exporter:v1.10.2
* quay.io/prometheus/prometheus:v3.5.3
* docker.io/grafana/grafana:13.0.2
* docker.io/library/caddy:2.11.4

---------



****************
AWS Preparation
****************

1. Created **AWS EC2 instance** ``monitoring``
2. Created **Security Group** ``sg-<aws-sg-id> - ssh``
3. Added **Inbound rule** ``22/tcp``, **Source** ``My IP`` to **SG** ``sg-<aws-sg-id> - ssh``
4. Created **SG** ``sg-<aws-sg-id> - monitoring-web``
5. Added **Inbound rules** ``80/tcp``, ``443/tcp``, **Source** ``0.0.0.0/0`` to **SG** ``sg-<aws-sg-id> - monitoring-web``
6. Applied **SG** ``sg-<aws-sg-id> - monitoring-web`` and ``sg-<aws-sg-id> - ssh`` to instance ``monitoring``
7. Created **Elastic IP** ``<aws-eip>``
8. Associated **EIP** ``<aws-eip>`` with instance ``monitoring``
9. Created **Public hosted zone** ``example.com`` in **Route 53**
10. Configured **AWS Route 53** nameservers, ``ns-1.awsdns.com.`` ``ns-22.awsdns.co.uk.`` ``ns-333.awsdns.org.`` ``ns-4444.awsdns.net.`` for domain ``example.com`` in the registrar's control panel
11. Created record **A** ``monitoring.example.com`` with value ``<aws-eip>`` in **Route 53**

---------



*******************************************
System Preparation (as sudoer ec2-user)
*******************************************

1. Connected to instance ``monitoring``, ``ssh ec2-user@<aws-eip>``
2. Registered the system with **RHSM**: ``$ sudo subscription-manager register``
3. Updated the system: ``$ sudo dnf upgrade``
4. Installed package ``systemd-container-257-23.el10_2.2.x86_64``: ``$ sudo dnf install systemd-container``
5. Installed package ``skopeo-1.22.2-1.el10_2.x86_64``: ``$ sudo dnf install skopeo``
6. Added the ``quay.io`` registry to the file :file:`/etc/containers/registries.conf`
7. Created user ``monitoring``: ``$ sudo useradd -m -s /bin/bash monitoring``
8. Enabled ``linger``: ``$ sudo loginctl enable-linger monitoring``
9. Allowed unprivileged processes to bind ports >= 80

.. code-block::

   $ sudo tee /etc/sysctl.d/90-unprivileged-ports.conf <<'EOF'
   net.ipv4.ip_unprivileged_port_start=80
   EOF

.. note::

   This allows the rootless Caddy container to bind ports 80 and 443 without root privileges.

10. Applied the changes: ``$ sudo sysctl --system``
11. subuid/subgid configured for the user: ``$ grep monitoring /etc/subuid /etc/subgid``

---------



*******************
Directory Structure
*******************

1. Opened an interactive shell session as user ``monitoring``: ``$ sudo machinectl shell monitoring@.host`` (command executed as **sudoer** user ``ec2-user``)
2. Next, as user ``monitoring``, created the directory structure

.. code-block::

   $ mkdir -p ~/.config/containers/systemd
   $ mkdir -p ~/config/prometheus
   $ mkdir -p ~/config/grafana/provisioning/datasources
   $ mkdir -p ~/config/caddy

---------



*******************
Configuration Files
*******************

Commands executed as user ``monitoring``

1. Created the file :file:`~/config/prometheus/prometheus.yml` with the following contents:

.. code-block::

   global:
     scrape_interval: 15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: prometheus
       static_configs:
         - targets: ['localhost:9099']

     - job_name: node
       static_configs:
         - targets:
             - 'host.containers.internal:9100'

       metric_relabel_configs:
         # root mount
         - source_labels: [mountpoint]
           regex: ^/host/root$
           target_label: mountpoint
           replacement: /

         # remaining mountpoints
         - source_labels: [mountpoint]
           regex: ^/host/root(/.*)$
           target_label: mountpoint
           replacement: '$1'

2. Created the file :file:`~/config/grafana/grafana.env`

.. code-block::

   GF_SECURITY_ADMIN_USER=monitoring_root
   GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana-admin-password
   GF_SERVER_ROOT_URL=https://monitoring.example.com
   GF_SERVER_DOMAIN=monitoring.example.com
   GF_ANALYTICS_REPORTING_ENABLED=false
   GF_ANALYTICS_CHECK_FOR_UPDATES=false

3. Created the file :file:`~/config/grafana/provisioning/datasources/prometheus.yml`

.. code-block::

   apiVersion: 1

   datasources:
     - name: Prometheus
       type: prometheus
       access: proxy
       url: http://prometheus:9099
       isDefault: true
       editable: false

4. Created a podman secret: ``printf '%s' 'superSecretPasswd' | podman secret create grafana-admin-password -``

5. Created the file :file:`~/config/caddy/Caddyfile`

.. code-block::

   monitoring.example.com {
       reverse_proxy grafana:3000
   }

---------



*******************
Quadlet Unit Files
*******************

Commands executed as user ``monitoring``

1. Created the file :file:`~/.config/containers/systemd/monitoring.network` with the following contents:

.. code-block::

   [Network]
   NetworkName=monitoring
   Driver=bridge

2. Created the file :file:`~/.config/containers/systemd/node-exporter.container`

.. code-block::

   [Unit]
   Description=Node Exporter
   After=network-online.target

   [Container]
   Image=quay.io/prometheus/node-exporter:v1.10.2
   ContainerName=node-exporter
   Network=host

   # Mount host proc/sys/root for metrics collection
   Volume=/proc:/host/proc:ro
   Volume=/sys:/host/sys:ro
   Volume=/:/host/root:ro,rbind

   # SELinux: disable label confinement, otherwise SELinux will block proc/sys
   SecurityLabelDisable=true

   Exec=--web.listen-address=0.0.0.0:9100 \
        --path.procfs=/host/proc \
        --path.sysfs=/host/sys \
        --path.rootfs=/host/root \
        --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run|var/lib/containers|home/[^/]+/\.local/share/containers)($|/)

   ReadOnly=true

   [Service]
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=default.target

3. Created the file :file:`~/.config/containers/systemd/prometheus-data.volume`

.. code-block::

   [Volume]
   VolumeName=prometheus-data

4. Created the file :file:`~/.config/containers/systemd/prometheus.container`

.. code-block::

   [Unit]
   Description=Prometheus
   Requires=monitoring-network.service
   Wants=node-exporter.service
   After=network-online.target monitoring-network.service node-exporter.service

   [Container]
   Image=quay.io/prometheus/prometheus:v3.5.3
   ContainerName=prometheus
   Network=monitoring

   # Named volume (Podman sets the SELinux label automatically)
   Volume=prometheus-data:/prometheus

   # Config bind mount — requires :Z for SELinux relabeling
   Volume=%h/config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro,Z

   Exec=--config.file=/etc/prometheus/prometheus.yml \
        --storage.tsdb.path=/prometheus \
        --storage.tsdb.retention.time=30d \
        --web.listen-address=0.0.0.0:9099

   [Service]
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=default.target

5. Created the file :file:`~/.config/containers/systemd/grafana-data.volume`

.. code-block::

   [Volume]
   VolumeName=grafana-data

6. Created the file :file:`~/.config/containers/systemd/grafana.container`

.. code-block::

   [Unit]
   Description=Grafana
   Requires=monitoring-network.service
   Wants=prometheus.service
   After=network-online.target monitoring-network.service prometheus.service

   [Container]
   Image=docker.io/grafana/grafana:13.0.2
   ContainerName=grafana
   Network=monitoring

   # Named volume for persistent data
   Volume=grafana-data:/var/lib/grafana

   # Secret
   Secret=grafana-admin-password

   # Datasource provisioning
   Volume=%h/config/grafana/provisioning:/etc/grafana/provisioning:ro,Z

   EnvironmentFile=%h/config/grafana/grafana.env

   [Service]
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=default.target

7. Created the file :file:`~/.config/containers/systemd/caddy-config.volume`

.. code-block::

   [Volume]
   VolumeName=caddy-config

8. Created the file :file:`~/.config/containers/systemd/caddy-data.volume`

.. code-block::

   [Volume]
   VolumeName=caddy-data

9. Created the file :file:`~/.config/containers/systemd/caddy.container`

.. code-block::

   [Unit]
   Description=Caddy Reverse Proxy
   Requires=monitoring-network.service grafana.service
   After=network-online.target monitoring-network.service grafana.service

   [Container]
   Image=docker.io/library/caddy:2.11.4
   ContainerName=caddy
   Network=monitoring

   # Publish ports 80 and 443 (HTTP + HTTPS + HTTP/3 QUIC)
   PublishPort=0.0.0.0:80:80
   PublishPort=0.0.0.0:443:443
   PublishPort=0.0.0.0:443:443/udp

   # Named volumes for TLS certificates and Caddy config
   Volume=caddy-data:/data
   Volume=caddy-config:/config

   # Caddyfile
   Volume=%h/config/caddy/Caddyfile:/etc/caddy/Caddyfile:ro,Z

   [Service]
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=default.target

---------



*******
Startup
*******

1. Reloaded the user **systemd** service configuration: ``$ systemctl --user daemon-reload``
2. Verified that **quadlet** generated the units: ``$ systemctl --user list-unit-files | grep -E 'node-exporter|prometheus|grafana|caddy|monitoring'``
3. Started all services

.. code-block::

   $ systemctl --user start node-exporter.service
   $ systemctl --user start monitoring-network.service # in newer Quadlet versions the unit is usually named after the file with the -network.service suffix.
   $ systemctl --user start prometheus.service
   $ systemctl --user start grafana.service
   $ systemctl --user start caddy.service

---------



**********************
First Login to Grafana
**********************

1. https://monitoring.example.com/login
**Login**: the username is ``GF_SECURITY_ADMIN_USER`` from the file :file:`~/config/grafana/grafana.env`, the password comes from ``podman secret grafana-admin-password``
2. The Prometheus datasource is already configured via provisioning (/config/grafana/provisioning/datasources/prometheus.yml)
3. Dashboard imported: Node Exporter Full: ID 1860

---------



************
Verification
************

All systemd units for the monitoring services exist and are registered.
==========================================================================

.. code-block::

   $ systemctl --user list-unit-files | grep -E 'prometheus|grafana|caddy|node|monitoring'
   caddy-config-volume.service             generated -
   caddy-data-volume.service               generated -
   caddy.service                           generated -
   grafana-data-volume.service             generated -
   grafana.service                         generated -
   monitoring-network.service              generated -
   node-exporter.service                   generated -
   prometheus-data-volume.service          generated -
   prometheus.service                      generated -

---------

Status of all Quadlet objects: containers are running, volumes and network are created.
===========================================================================================

.. code-block::

   $ podman quadlet list
   NAME                     UNIT NAME                       PATH ON DISK                                                         STATUS          APPLICATION
   caddy-config.volume      caddy-config-volume.service     /home/monitoring/.config/containers/systemd/caddy-config.volume      inactive/dead
   caddy-data.volume        caddy-data-volume.service       /home/monitoring/.config/containers/systemd/caddy-data.volume        inactive/dead
   caddy.container          caddy.service                   /home/monitoring/.config/containers/systemd/caddy.container          active/running
   grafana-data.volume      grafana-data-volume.service     /home/monitoring/.config/containers/systemd/grafana-data.volume      inactive/dead
   grafana.container        grafana.service                 /home/monitoring/.config/containers/systemd/grafana.container        active/running
   monitoring.network       monitoring-network.service      /home/monitoring/.config/containers/systemd/monitoring.network       active/exited
   node-exporter.container  node-exporter.service           /home/monitoring/.config/containers/systemd/node-exporter.container  active/running
   prometheus-data.volume   prometheus-data-volume.service  /home/monitoring/.config/containers/systemd/prometheus-data.volume   inactive/dead
   prometheus.container     prometheus.service              /home/monitoring/.config/containers/systemd/prometheus.container     active/running

.. note::

   inactive/dead for volume units is normal.
   Volumes are not long-running services — they are created once and remain.

---------

All 4 containers (node-exporter, prometheus, grafana, caddy) are running.
=============================================================================

.. code-block::

   $ podman container list
   CONTAINER ID  IMAGE                                     COMMAND               CREATED        STATUS        PORTS                                                                     NAMES
   f29a798eb867  quay.io/prometheus/node-exporter:v1.10.2                        3 minutes ago  Up 3 minutes  9100/tcp                                                                  node-exporter
   2bfe5a2bc368  quay.io/prometheus/prometheus:v3.5.3      --config.file=/et...  3 minutes ago  Up 3 minutes  9090/tcp                                                                  prometheus
   979595b3afa5  docker.io/grafana/grafana:13.0.2                                3 minutes ago  Up 3 minutes  3000/tcp                                                                  grafana
   0715daf22b4b  docker.io/library/caddy:2.11.4            caddy run --confi...  3 minutes ago  Up 3 minutes  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:443->443/udp, 2019/tcp  caddy

.. note::

   podman only shows the container's internal port. Forwarding of 9099 to the host is not configured.

   .. code-block::

      $ podman exec -it prometheus sh
      /prometheus $ netstat -lntp | grep 9099
      tcp        0      0 :::9099                 :::*                    LISTEN      1/prometheus

---------

Each service is active and running without errors.
========================================================

.. code-block::

   $ systemctl --user status prometheus grafana caddy node-exporter --no-pager
   ● prometheus.service - Prometheus
        Loaded: loaded (/home/monitoring/.config/containers/systemd/prometheus.container; generated)
        Active: active (running) since Thu 2026-06-11 11:36:58 UTC; 4min 32s ago
    Invocation: 31644157fc274db78ed9ebe3442eed4a
      Main PID: 6102 (conmon)
         Tasks: 9 (limit: 8884)
        Memory: 73.7M (peak: 77.3M)
           CPU: 685ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/prometheus.service
                ├─libpod-payload-2bfe5a2bc368c1cf3317c4acacd2b70daea0fcd4f0be6e35bfa7c28ff232032f
                │ └─6104 /bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --storage.tsdb.retention.time=30d --web.listen-address=0.0.0.0:9099
                └─runtime
                  └─6102 /usr/bin/conmon --api-version 1 -c 2bfe5a2bc368c1cf3317c4acacd2b70daea0fcd4f0be6e35bfa7c28ff232032f -u 2bfe5a2bc368c1cf3317c4acacd2b70daea0fcd4f0be6e35bfa7c28ff232032f -r /usr/bin/crun -b /home/monitoring/.local/sha…

   ● grafana.service - Grafana
        Loaded: loaded (/home/monitoring/.config/containers/systemd/grafana.container; generated)
        Active: active (running) since Thu 2026-06-11 11:37:04 UTC; 4min 26s ago
    Invocation: 4032551d7b3441fc822d61c7e3cee7dd
      Main PID: 6147 (conmon)
         Tasks: 29 (limit: 8884)
        Memory: 412.2M (peak: 430.4M)
           CPU: 8.446s
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/grafana.service
                ├─libpod-payload-979595b3afa5cd65e0733cd90f2de04df8683247cbefeeb3f99d4a0b0701e543
                │ ├─6149 grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console cfg:default.paths.data=/var/lib/grafana cfg:default.paths.logs=/var/log/grafana cfg:defa…
                │ └─6233 /usr/share/grafana/data/plugins-bundled/elasticsearch/gpx_grafana_elasticsearch_datasource_linux_amd64
                └─runtime
                  └─6147 /usr/bin/conmon --api-version 1 -c 979595b3afa5cd65e0733cd90f2de04df8683247cbefeeb3f99d4a0b0701e543 -u 979595b3afa5cd65e0733cd90f2de04df8683247cbefeeb3f99d4a0b0701e543 -r /usr/bin/crun -b /home/monitoring/.local/sha…

   ● caddy.service - Caddy Reverse Proxy
        Loaded: loaded (/home/monitoring/.config/containers/systemd/caddy.container; generated)
        Active: active (running) since Thu 2026-06-11 11:37:09 UTC; 4min 21s ago
    Invocation: 6051d1ac2e554a5cae8386ee5b768e05
      Main PID: 6221 (conmon)
         Tasks: 22 (limit: 8884)
        Memory: 31.4M (peak: 59.4M)
           CPU: 831ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/caddy.service
                ├─libpod-payload-0715daf22b4b34f771948ee9f0d12a465f80dc4942324c41724274ec37ba835c
                │ └─6223 caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
                └─runtime
                  ├─6194 rootlessport
                  ├─6200 rootlessport-child
                  └─6221 /usr/bin/conmon --api-version 1 -c 0715daf22b4b34f771948ee9f0d12a465f80dc4942324c41724274ec37ba835c -u 0715daf22b4b34f771948ee9f0d12a465f80dc4942324c41724274ec37ba835c -r /usr/bin/crun -b /home/monitoring/.local/sha…

   ● node-exporter.service - Node Exporter
        Loaded: loaded (/home/monitoring/.config/containers/systemd/node-exporter.container; generated)
        Active: active (running) since Thu 2026-06-11 11:36:45 UTC; 4min 45s ago
    Invocation: dfb967a05e4d469fa874aeccfe535954
      Main PID: 6058 (conmon)
         Tasks: 6 (limit: 8884)
        Memory: 10.5M (peak: 20.1M)
           CPU: 390ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/node-exporter.service
                ├─libpod-payload-f29a798eb86731e41ddf4d90c179d9278b247d9b1bcdc2a4f5e11e9e94236845
                │ └─6060 /bin/node_exporter
                └─runtime
                  └─6058 /usr/bin/conmon --api-version 1 -c f29a798eb86731e41ddf4d90c179d9278b247d9b1bcdc2a4f5e11e9e94236845 -u f29a798eb86731e41ddf4d90c179d9278b247d9b1bcdc2a4f5e11e9e94236845 -r /usr/bin/crun -b /home/monitoring/.local/sha…

---------

Grafana responds, the database is OK, the version is correct.
===================================================================

.. code-block::

   $ curl -k https://monitoring.example.com/api/health
   {
     "database": "ok",
     "version": "13.0.2",
     "commit": "3fcdbc5a"
   }

---------

HTTPS works, Caddy proxies requests, redirect to /login
=============================================================

.. code-block::

   $ curl -I https://monitoring.example.com
   HTTP/2 302
   alt-svc: h3=":443"; ma=2592000
   cache-control: no-store
   content-type: text/html; charset=utf-8
   date: Thu, 11 Jun 2026 11:42:33 GMT
   location: /login
   via: 1.1 Caddy
   x-content-type-options: nosniff
   x-frame-options: deny
   x-xss-protection: 1; mode=block

---------

Startup order is correct: grafana depends on prometheus and the network.
==============================================================================

.. code-block::

   $ systemctl --user list-dependencies grafana
   grafana.service
   ● ├─app.slice
   ● ├─monitoring-network.service
   ● ├─podman-user-wait-network-online.service
   ● ├─prometheus.service
   ● └─basic.target
   ●   ├─systemd-tmpfiles-setup.service
   ●   ├─paths.target
   ●   ├─sockets.target
   ●   │ └─dbus.socket
   ●   └─timers.target
   ●     ├─grub-boot-success.timer
   ●     └─systemd-tmpfiles-clean.timer

---------

node-exporter is available on port 9100 and serves host metrics.
======================================================================

.. code-block::

   $ curl -s http://localhost:9100/metrics | head
   # HELP go_gc_duration_seconds A summary of the wall-time pause (stop-the-world) duration in garbage collection cycles.
   # TYPE go_gc_duration_seconds summary
   go_gc_duration_seconds{quantile="0"} 1.9692e-05
   go_gc_duration_seconds{quantile="0.25"} 4.1778e-05
   go_gc_duration_seconds{quantile="0.5"} 4.6841e-05
   go_gc_duration_seconds{quantile="0.75"} 4.9412e-05
   go_gc_duration_seconds{quantile="1"} 8.1833e-05
   go_gc_duration_seconds_sum 0.001962299
   go_gc_duration_seconds_count 41
   # HELP go_gc_gogc_percent Heap size target percentage configured by the user, otherwise 100. This value is set by the GOGC environment variable, and the runtime/debug.SetGCPercent function. Sourced from /gc/gogc:percent.

---------

Verifying the solution's health after a reboot.
=====================================================

.. code-block::

   $ sudo systemctl reboot
   $ ssh ec2-user@<aws-eip>
   $ sudo machinectl shell monitoring@.host
   $ systemctl --user status prometheus grafana caddy node-exporter --no-pager
   ● prometheus.service - Prometheus
        Loaded: loaded (/home/monitoring/.config/containers/systemd/prometheus.container; generated)
        Active: active (running) since Thu 2026-06-11 13:26:57 UTC; 1min 37s ago
    Invocation: 40b99fdcc2284267ac903e2a4f5fd0df
      Main PID: 2107 (conmon)
         Tasks: 9 (limit: 8884)
        Memory: 180M (peak: 180.5M)
           CPU: 476ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/prometheus.service
                ├─libpod-payload-d4cb349c7cb514afc6758225967b606a8a70072441bfa7fc86274411c58dceba
                │ └─2112 /bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --storage.tsdb.retention.time=30d --web.listen-address=0.0.0.0:9099
                └─runtime
                  └─2107 /usr/bin/conmon --api-version 1 -c d4cb349c7cb514afc6758225967b606a8a70072441bfa7fc86274411c58dceba -u d4cb349c7cb514afc6758225967b606a8a70072441bfa7fc86274411c58dceba -r /usr/b…

   ● grafana.service - Grafana
        Loaded: loaded (/home/monitoring/.config/containers/systemd/grafana.container; generated)
        Active: active (running) since Thu 2026-06-11 13:26:57 UTC; 1min 37s ago
    Invocation: de84632836394ce9b5545c6d99691068
      Main PID: 2207 (conmon)
         Tasks: 24 (limit: 8884)
        Memory: 575.1M (peak: 576.8M)
           CPU: 3.765s
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/grafana.service
                ├─libpod-payload-02898ae1f6a26df3b03c42ada56e8960dfa77667cec769e3bf9c33a521931103
                │ ├─2209 grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console cfg:default.paths.data=/var/lib/grafana cfg:defaul…
                │ └─2467 /usr/share/grafana/data/plugins-bundled/elasticsearch/gpx_grafana_elasticsearch_datasource_linux_amd64
                └─runtime
                  └─2207 /usr/bin/conmon --api-version 1 -c 02898ae1f6a26df3b03c42ada56e8960dfa77667cec769e3bf9c33a521931103 -u 02898ae1f6a26df3b03c42ada56e8960dfa77667cec769e3bf9c33a521931103 -r /usr/b…

   ● caddy.service - Caddy Reverse Proxy
        Loaded: loaded (/home/monitoring/.config/containers/systemd/caddy.container; generated)
        Active: active (running) since Thu 2026-06-11 13:26:58 UTC; 1min 36s ago
    Invocation: 17f4d7a343704de184b70caf2ab16bbf
      Main PID: 2444 (conmon)
         Tasks: 21 (limit: 8884)
        Memory: 70M (peak: 70.5M)
           CPU: 256ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/caddy.service
                ├─libpod-payload-5b2436607a5af1219a15b4aad3df36254d2707057fdcafdf5b9fda754f21bb8e
                │ └─2446 caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
                └─runtime
                  ├─2341 rootlessport
                  ├─2393 rootlessport-child
                  └─2444 /usr/bin/conmon --api-version 1 -c 5b2436607a5af1219a15b4aad3df36254d2707057fdcafdf5b9fda754f21bb8e -u 5b2436607a5af1219a15b4aad3df36254d2707057fdcafdf5b9fda754f21bb8e -r /usr/b…

   ● node-exporter.service - Node Exporter
        Loaded: loaded (/home/monitoring/.config/containers/systemd/node-exporter.container; generated)
        Active: active (running) since Thu 2026-06-11 13:26:56 UTC; 1min 38s ago
    Invocation: e36ffbc13c0b43aab0a100dd0e4fe9f1
      Main PID: 1773 (conmon)
         Tasks: 6 (limit: 8884)
        Memory: 29.2M (peak: 39.1M)
           CPU: 278ms
        CGroup: /user.slice/user-1001.slice/user@1001.service/app.slice/node-exporter.service
                ├─libpod-payload-0419b8c4ca0e766e6178c4702479204387f498f2a2c38bb564b3232f36bdb631
                │ └─1775 /bin/node_exporter
                └─runtime
                  └─1773 /usr/bin/conmon --api-version 1 -c 0419b8c4ca0e766e6178c4702479204387f498f2a2c38bb564b3232f36bdb631 -u 0419b8c4ca0e766e6178c4702479204387f498f2a2c38bb564b3232f36bdb631 -r /usr/b…

The solution works correctly after a reboot.
