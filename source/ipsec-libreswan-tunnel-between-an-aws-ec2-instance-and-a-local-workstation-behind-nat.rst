#####################################################################################
IPsec Libreswan Tunnel Between an AWS EC2 Instance and a Local Workstation Behind NAT
#####################################################################################

************
Introduction
************

The objective is to establish a secure tunnel between a workstation located behind NAT and a cloud server with a public address, without using a pre-shared key (PSK). Instead, authentication is based on a per-host RSA key pair.

Libreswan was chosen as the implementation, being the reference IPsec implementation for RHEL-family systems. The scheme is asymmetric with respect to roles: the local machine acts as the connection initiator, since its address is unstable and unknown in advance, while the cloud host, which has a permanent public address, operates in listening mode and accepts connections from any source. All IPsec traffic is encapsulated in NAT-T (UDP 4500), which removes the need to open ESP (IP protocol 50) at the network filtering level.

Below is a step-by-step description, from environment preparation and key generation to the final verification of the established tunnel on both hosts.

There are two peer hosts.

The hosts are designated ``left`` and ``right``.

1. Local workstation (RHEL 9) behind NAT — host ``left``
2. AWS EC2 instance (RHEL 10) with an AWS Elastic IP — host ``right``

---------



************
Architecture
************

.. graphviz::

   digraph ipsec{
      rankdir=TB;
      newrank=true;
      compound=true;
      labelloc="t";
      ranksep=1;
      fontname="monospace";
      label="Encrypted IPsec Libreswan Host-to-Host Tunnel Deployment Architecture\n(ESP over NAT-T)";
      node[fontname="monospace"];
      edge[fontname="monospace"];
      hostleft[label="Host left\n192.168.1.8/32\nInitiator\nauto=start\nleft=%defaultroute\nauthby=rsasig"];
      router[label="Router\nNAT"];
      internet[label="Internet"];
      subgraph cluster_aws{
         label="Amazon Web Services";
         eip[label="Elastic IP"];
         sg[label="Security Group"];
         hostright[label="Host right\n<right-private-ipv4>/32\nResponder\nauto=add\nright=%any\nauthby=rsasig"];
      }
      hostleft->router->internet;
      {rank="same"; router; internet;}
      internet->eip[lhead=cluster_aws];
      eip->sg->hostright;
      {rank="same"; eip; sg;}
   }

---------



***********
Environment
***********

Host ``left``
=============

* Red Hat Enterprise Linux release 9.7 (Plow)
* kernel-5.14.0-611.55.1.el9_7.x86_64
* libreswan-4.15-8.el9.x86_64

---------

Host ``right``
==============

* Red Hat Enterprise Linux release 10.1 (Coughlan)
* kernel-6.12.0-124.56.1.el10_1.x86_64
* libreswan-5.3-1.el10.x86_64

---------

Tunnel Configuration
=====================

==============   ===========
IKE version      IKEv2
Authentication   Raw RSA key
==============   ===========

---------



****************
AWS Preparation
****************

Security and Access Control Configuration
==========================================

1. Imported **Key Pair** ``default``, **Type** ``ed25519``
2. Created **Security Group** ``sg-<aws-sg-id> - ssh``
3. Added **Inbound rule** ``22/tcp``, **Source** ``My IP`` to **SG** ``sg-<aws-sg-id> - ssh``
4. Created **SG** ``sg-<aws-sg-id> - udp-500-4500``
5. Added **Inbound rules** ``4500/udp`` and ``500/udp``, **Source** ``0.0.0.0/0`` to **SG** ``sg-<aws-sg-id> - udp-500-4500``

.. note::

   ESP (IP protocol 50) was not opened — the topology includes NAT, so IPsec traffic is encapsulated in NAT-T (UDP 4500). The AWS EIP implements static bidirectional NAT, with a single public address permanently mapped to a single private instance address.

---------

Deploying an EC2 Instance with Configured Access
=================================================

1. Created a **t3.small** instance ``right`` running RHEL 10
2. Applied **SG** ``sg-<aws-sg-id> - ssh`` and ``sg-<aws-sg-id> - udp-500-4500`` to instance ``right``

---------

Allocating and Associating an EIP
==================================

1. Created **Elastic IP address** ``<right-public-ipv4>``
2. Associated **EIP** ``<right-public-ipv4>`` with ``right``

---------



*****************
Host Preparation
*****************

1. Host ``right`` registered with RHSM: ``$ sudo subscription-manager register``
2. Host ``left`` configured with a static IPv4 address via NetworkManager

.. code-block::

   $ sudo nmcli connection modify <connection-name> ipv4.addresses 192.168.1.8/24
   $ sudo nmcli connection modify <connection-name> ipv4.gateway 192.168.1.1
   $ sudo nmcli connection modify <connection-name> ipv4.dns "1.1.1.1 1.0.0.1"
   $ sudo nmcli connection modify <connection-name> ipv4.method manual
   $ sudo nmcli connection up <connection-name>

3. Both hosts were updated; after the ``kernel`` update the systems were rebooted.

.. code-block::

   $ sudo dnf upgrade
   $ sudo systemctl reboot

---------



**************************************
Installing and Initializing Libreswan
**************************************

1. Performed on both hosts

.. code-block::

   $ sudo dnf install libreswan
   $ sudo ipsec initnss
   $ sudo systemctl enable --now ipsec

2. On host ``left``, the ``ipsec`` service was added to **firewalld**

.. code-block::

   $ sudo firewall-cmd --permanent --add-service="ipsec"
   $ sudo firewall-cmd --reload

.. note::

   On host ``right``, no **host-based firewall** is used; filtering is performed at the **SG** level.

---------



*********************************
Key Generation and Identifiers
*********************************

1. On each host, an **RSA key** pair was generated and stored in the **NSS database**: ``$ sudo ipsec newhostkey``
2. Retrieved the CKAIDs of both hosts: ``$ sudo ipsec showhostkey --list``
3. Extracted the public keys in the format required for the configuration file

.. code-block::

   # on host left
   $ sudo ipsec showhostkey --left --ckaid <ckaid_of_left_host>

   # on host right
   $ sudo ipsec showhostkey --right --ckaid <ckaid_of_right_host>

---------



********************
Tunnel Configuration
********************

Host ``left`` is the connection initiator (``auto=start``). Host ``right`` waits for an incoming connection (``auto=add``) and accepts the connection from any address (``right=%any``), since the actual IP address of ``left`` is behind NAT, unknown in advance, and subject to change.

1. Created the configuration file :file:`/etc/ipsec.d/workstation-aws.conf` on host ``left`` with the following contents:

.. code-block::

   conn workstation-aws
      auto=start    # initiator, brings up the tunnel on ipsec start
      authby=rsasig

      # Peer A (host left)
      left=%defaultroute    # behind NAT, IP determined automatically
      leftid=@peer_a
      leftsubnet=192.168.1.8/32
      leftrsasigkey=<ckaid>

      # Peer B (host right)
      right=<right-public-ipv4>
      rightid=@peer_b
      rightsubnet=<right-private-ipv4>/32
      rightrsasigkey=<ckaid>

2. Created the configuration file :file:`/etc/ipsec.d/workstation-aws.conf` on host ``right``

.. code-block::

   conn workstation-aws
      auto=add    # add, since right=%any, responder, waits for incoming IKE
      authby=rsasig

      # Peer B (host right)
      left=%defaultroute
      leftid=@peer_b
      leftsubnet=<right-private-ipv4>/32
      leftrsasigkey=<ckaid>

      # Peer A (host left)
      right=%any    # accept from any address
      rightid=@peer_a
      rightsubnet=192.168.1.8/32
      rightrsasigkey=<ckaid>

3. After creating the configuration files on both hosts, the ``ipsec`` service was restarted: ``$ sudo systemctl restart ipsec``

---------



************
Verification
************

Host ``left``
=============

Connectivity Through the Tunnel
--------------------------------

.. code-block::

   # left → right
   $ ping -c 3 <right-private-ipv4>
   PING <right-private-ipv4> (<right-private-ipv4>) 56(84) bytes of data.
   64 bytes from <right-private-ipv4>: icmp_seq=1 ttl=64 time=175 ms
   64 bytes from <right-private-ipv4>: icmp_seq=2 ttl=64 time=174 ms
   64 bytes from <right-private-ipv4>: icmp_seq=3 ttl=64 time=175 ms

   --- <right-private-ipv4> ping statistics ---
   3 packets transmitted, 3 received, 0% packet loss, time 2002ms
   rtt min/avg/max/mdev = 173.818/174.492/175.137/0.538 ms

---------

Traffic is Flowing Through the Tunnel
---------------------------------------

.. code-block::

   $ sudo ipsec whack --trafficstatus
   006 #2: "workstation-aws", type=ESP, add_time=1778683704, inBytes=15624, outBytes=15708, maxBytes=2^63B, id='@peer_b'

---------

Tunnel is Up
------------

.. code-block::

   $ sudo ipsec status
   000 Total IPsec connections: loaded 1, active 1
   000 #1: "workstation-aws":4500 STATE_V2_ESTABLISHED_IKE_SA (established IKE SA); REKEY in 27237s; REPLACE in 27955s; newest; idle;
   000 #2: "workstation-aws":4500 STATE_V2_ESTABLISHED_CHILD_SA (established Child SA); REKEY in 27167s; REPLACE in 27955s; newest; eroute owner; IKE SA #1; idle;
   000 #2: "workstation-aws" esp.eca64054@<right-public-ipv4> esp.9484d67@192.168.1.8 tun.0@<right-public-ipv4> tun.0@192.168.1.8 Traffic: ESPin=15KB ESPout=15KB ESPmax=2^63B

---------

Traffic is Encrypted at the Packet Level
-------------------------------------------

.. code-block::

   $ sudo tcpdump -i any udp port 4500
   20:16:26.026236 wlp2s0 In  IP <right-hostname>.compute-1.amazonaws.com.ipsec-nat-t > <left-hostname>.ipsec-nat-t: UDP-encap: ESP(spi=0x58360cf4,seq=0x14), length 120
   20:16:26.026449 wlp2s0 Out IP <left-hostname>.ipsec-nat-t > <right-hostname>.compute-1.amazonaws.com.ipsec-nat-t: UDP-encap: ESP(spi=0xf7190c81,seq=0x14), length 120

---------

SAs are Loaded into the Kernel
--------------------------------

.. code-block::

   $ sudo ip xfrm state
   src <right-public-ipv4> dst 192.168.1.8
      proto esp spi 0x58360cf4 reqid 16389 mode tunnel
      replay-window 0 flag af-unspec esn
      aead rfc4106(gcm(aes)) 0x... 128
      encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
      lastused 2026-05-13 20:22:57
      anti-replay esn context:
         seq-hi 0x0, seq 0x19, oseq-hi 0x0, oseq 0x0
         replay_window 128, bitmap-length 4
         00000000 00000000 00000000 01ffffff
   src 192.168.1.8 dst <right-public-ipv4>
      proto esp spi 0xf7190c81 reqid 16389 mode tunnel
      replay-window 0 flag af-unspec esn
      aead rfc4106(gcm(aes)) 0x... 128
      encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
      lastused 2026-05-13 20:22:57
      anti-replay esn context:
         seq-hi 0x0, seq 0x0, oseq-hi 0x0, oseq 0x19
         replay_window 128, bitmap-length 4
         00000000 00000000 00000000 00000000

---------

Host ``right``
==============

Connectivity Through the Tunnel
--------------------------------

.. code-block::

   # right → left
   $ ping -c 3 192.168.1.8
   PING 192.168.1.8 (192.168.1.8) 56(84) bytes of data.
   64 bytes from 192.168.1.8: icmp_seq=1 ttl=64 time=174 ms
   64 bytes from 192.168.1.8: icmp_seq=2 ttl=64 time=199 ms
   64 bytes from 192.168.1.8: icmp_seq=3 ttl=64 time=206 ms

   --- 192.168.1.8 ping statistics ---
   3 packets transmitted, 3 received, 0% packet loss, time 2002ms
   rtt min/avg/max/mdev = 174.087/193.080/206.139/13.741 ms

---------

Traffic is Flowing Through the Tunnel
---------------------------------------

.. code-block::

   $ sudo ipsec whack --trafficstatus
   #2: "workstation-aws"[1] <left-public-ipv4>, type=ESP, add_time=1778683704, inBytes=15708, outBytes=15708, maxBytes=2^63B, id='@peer_a'

---------

Tunnel is Up
------------

.. code-block::

   $ sudo ipsec status
   Total IPsec connections: loaded 2, routed 1, active 1
   #1: "workstation-aws"[1] <left-public-ipv4>:4500 ESTABLISHED_IKE_SA (established IKE SA); REKEY in 27322s; REPLACE in 27592s; newest; idle;
   #2: "workstation-aws"[1] <left-public-ipv4>:4500 ESTABLISHED_CHILD_SA (established Child SA); REKEY in 27322s; REPLACE in 27592s; newest; eroute owner; IKE SA #1; idle;
   #2: "workstation-aws"[1] <left-public-ipv4> esp.9484d67@<left-public-ipv4> esp.eca64054@<right-private-ipv4> tun.0@<left-public-ipv4> tun.0@<right-private-ipv4> Traffic: ESPin=15KB ESPout=15KB ESPmax=2^63B

---------

Traffic is Encrypted at the Packet Level
-------------------------------------------

.. code-block::

   $ sudo tcpdump -i any udp port 4500
   15:22:55.894165 ens5  In  IP <left-public-ipv4>.ipsec-nat-t > <right-private-ipv4>.ec2.internal.ipsec-nat-t: UDP-encap: ESP(spi=0xf7190c81,seq=0x17), length 120
   15:22:55.894221 ens5  Out IP <right-private-ipv4>.ec2.internal.ipsec-nat-t > <left-public-ipv4>.ipsec-nat-t: UDP-encap: ESP(spi=0x58360cf4,seq=0x17), length 120

---------

SAs are Loaded into the Kernel
--------------------------------

.. code-block::

   $ sudo ip xfrm state
   src <right-private-ipv4> dst <left-public-ipv4>
      proto esp spi 0x58360cf4 reqid 16397 mode tunnel
      replay-window 0 flag af-unspec esn
      aead rfc4106(gcm(aes)) 0x... 128
      encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
      lastused 2026-05-13 15:22:57
      anti-replay esn context:
         seq-hi 0x0, seq 0x0, oseq-hi 0x0, oseq 0x19
         replay_window 0, bitmap-length 0
        dir out
   src <left-public-ipv4> dst <right-private-ipv4>
      proto esp spi 0xf7190c81 reqid 16397 mode tunnel
      replay-window 0 flag af-unspec esn
      aead rfc4106(gcm(aes)) 0x... 128
      encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
      lastused 2026-05-13 15:22:57
      anti-replay esn context:
         seq-hi 0x0, seq 0x19, oseq-hi 0x0, oseq 0x0
         replay_window 128, bitmap-length 4
         00000000 00000000 00000000 01ffffff

---------

``IKE SA`` is established, ``Child SA`` is active, ``ESP`` traffic is encapsulated in ``UDP 4500``, ``SAs`` are loaded into the kernel on both hosts — the tunnel is working correctly.
