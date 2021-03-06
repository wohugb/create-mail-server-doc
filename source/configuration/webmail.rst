Webmail
===========

启用Web访问
------------------

You may need to enable web access in the firewall. Check the firewall configuration if this neccessary.

另类: SquirrelMail
---------------------------------

This howto in previous editions used to have SquirrelMail as the webmail client. It is more mature with a longer testing record. It has a large library of various plugins. Please read the SquirrelMail extension further down on how to install it instead if prefered.

Roundcube Webmail客户端
-----------------------------

To install Roundcube
sudo aptitude install roundcube roundcube-mysql roundcube-plugins
It will ask you if you want to configure its database access, answer yes, then select mysql. Then it will ask for the root mysql uses password, which it will create a roundcube mysql user and ask for its desired password.
This will create a symblink in /etc/apache2/conf.d/ to /etc/roundcube/apache.conf. Edit this file.

.. code-block:: sh

   sudo vi /etc/roundcube/apache.conf

Depending on your setup you may want to move those Alias commands at the top to your virtual hosts configuration, or for this example enable them here for all hosts.

.. code-block:: apache

   # Uncomment them to use it or adapt them to your configuration
   Alias /roundcube/program/js/tiny_mce/ /usr/share/tinymce/www/
   Alias /roundcube /var/lib/roundcube

Next edit the configuration file

.. code-block:: sh

   sudo vi /etc/roundcube/main.inc.php
   
Modify these lines for added security and ease of log in:

.. code-block:: php

   <? php
   $rcmail_config['default_host'] = 'ssl://localhost:993';
   
   $rcmail_config['smtp_server'] = 'ssl://localhost';
   
   $rcmail_config['smtp_port'] = 465;
   
   # keep as default or change to your mail server name
   $rcmail_config['smtp_helo_host'] = 'mail.example.com';
   
   $rcmail_config['create_default_folders'] = TRUE;

There are other tweaks and security features you can enable such as:

.. code-block:: php

   <? php
   $rcmail_config['sendmail_delay'] = 1;
   
But perhaps concentrate on getting the basics working firtst...
Save, exit and reload Apache to enable these aliases for Roundcube to work

.. code-block:: sh

   :sudo /etc/init.d/apache2 reload
   
Then go to your roundcube installation depending where and how you modified those Aliases, e.g. at http://mail.example.com/roundcube.

That should be it

You can obviously modify and tweak further. One thing that may be usefull is to have the Roundcube Apache Alias on different virtual hosts, and configure username_domain in main.inc.php to append different email addresses, or configure the default_host to different mail server depending on virtual host... More details on the Roundcube Wiki.
