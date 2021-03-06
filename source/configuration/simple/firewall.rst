防火墙
=========

Shorewall
-----------

Not essential for an EC2 image. It is essential for a normal server. UFW is bundled with recent Ubuntu distributions, but I still prefer Shorewall for servers.

安装
^^^^^^^^^^^^^^^^^

.. code-block:: sh

   sudo aptitude install shorewall shorewall-doc
   # for earlier ubuntu versions use package shorewall-common shorewall-perl shorewall-doc instead

Amazon provides a firewall/ access control for its servers, so not always needed then, but nice to have. And in all others situations; a must have.

配置
^^^^^^^^^^^^^^^^^

Basically at first you want to only allow SSH. Then SMTP and IMAP from your IP only.

When you are confident that the mail server is secure, you can open SMTP to the world. If you prefer you can also open IMAP to the world, unless you have a very small client IP range.

Later you may open web access to the webmail and admin gui. This you may also restrict to specific IPs.

SSH
^^^^^^^^^^^^^^^^^

By default Shorewall in Ubuntu has an empty set up. You can find the default values for Shorewall in /usr/share/doc/shorwall/default-config. And examples in /usr/share/doc/shorwall/examples. We will create a basic set up.

First configure which network adapters we are accessing the net.

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/interfaces /etc/shorewall/
   vi /etc/shorewall/interfaces
   net     eth0            detect          dhcp,tcpflags,logmartians,nosmurfs

Then we will configure network zones

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/zones /etc/shorewall/
   vi /etc/shorewall/zones

Add the firewall if not there and the internet as a zone.

.. code-block:: sh

   fw	firewall
   # loc 	ipv4		
   net     ipv4

Then if needed to specify hosts you can do it in this file. E.g. If you wanto specify what is your home IP etc.

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/hosts /etc/shorewall/
   vi /etc/shorewall/hosts
   # loc	eth0:192.168.0.0/24

Then set what is the default policy for firewall access.

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/policy /etc/shorewall/
   vi /etc/shorewall/policy
   $FW             net             ACCEPT
   net             $FW             DROP            info
   net             all             DROP            info
   # The FOLLOWING POLICY MUST BE LAST
   all             all             REJECT          info

For safety in case it goes down.

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/routestopped /etc/shorewall/
   vi /etc/shorewall/routestopped
   eth0            0.0.0.0                 routeback

You may put in a netmask of your ip range if you are more concerned.

Now for the main firewall rules. You can find predetermined macro rules for Shorewall in /usr/share/shorewall.

.. code-block:: sh

   cp /usr/share/doc/shorewall/default-config/rules /etc/shorewall/
   vi /etc/shorewall/rules
   SSH/ACCEPT      net             $FW

Open for business

Once your server is working come back to this step and open up SMTP and Web access to others.

.. code-block:: sh

   vi /etc/shorewall/rules
   Ping/ACCEPT     net             $FW

   # Permit all ICMP traffic FROM the firewall TO the net zone
   ACCEPT          $FW             net             icmp

   # mail lines
   SMTP/ACCEPT     net             $FW
   SMTPS/ACCEPT    net             $FW
   Submission/ACCEPT       net             $FW
   IMAP/ACCEPT     net             $FW
   IMAPS/ACCEPT    net             $FW

   #web
   Web/ACCEPT      net             $FW

Firewall configuring is always risky business, as it is easy to lock yourself out. To test the setup syntax, run

shorewall check

Restart it with

.. code-block:: sh

   /etc/init.d/shorewall restart

Then to switch it on during boot:

.. code-block:: sh

   vi /etc/default/shorewall
   startup=1

For more details on IP Tables and Shorewall, look up its website.
