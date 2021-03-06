简单安装
========

文档说明
--------

引用自: http://www.pixelinx.com/2010/10/creating-a-mail-server-on-ubuntu-using-postfix-courier-ssltls-spamassassin-clamav-and-amavis/

更新: This guide has been updated to work with Ubuntu 12.04 LTS.

注释: this has been tested to work on the following versions of Ubuntu:

* Ubuntu 12.04
* Ubuntu 11.04
* Ubuntu 10.04
* Ubuntu 9.04

One of the most fragile and fragmented services I’ve had to configure on Ubuntu is a mail server. No matter which of the many guides I follow, each time I do it there’s always something not working.

This one is mostly for my benefit, but hopefully it’ll be useful to others, too. I’ve tried to make the guide easy to follow and as short as possible. Please comment if something isn’t clear.

Before we start, I have to give a huge amount of credit to Ivar Abrahamsen for his guide which is, by far, one of the best ones out there.

So let’s kick off…

我们将建立一个邮件服务器，由以下几部分组成:

* Postfix is the mail transfer agent (MTA) responsible for accepting new messages and storing them on your server as well as allowing authorised users to send e-mail.
* Courier sits in front of Postfix and provides an IMAP and POP3 interface for clients to connect to.
* SASL with SSL and TLS allows you to authenticate and communicate with the mail server securely.
* SpamAssassin will analyse your e-mails as they arrive and will filter out what it thinks is spam.
* ClamAV will scan e-mails for viruses before delivering it to your inbox.
* Amavis ties SpamAssasin and ClamAV together, and is itself hooked into Postfix.
* MySQL will be used to manage user accounts and e-mail forwarding.

安装
------

首先, 切换到跟用户.

.. code-block:: sh

   sudo su -

为简单起见， 我们将安装的所有软件一气呵成：


.. code-block:: sh

   apt-get update
   apt-get install -y mysql-server postfix postfix-mysql libsasl2-modules libsasl2-modules-sql libgsasl7 libauthen-sasl-cyrus-perl sasl2-bin libpam-mysql clamav-base libclamav6 clamav-daemon clamav-freshclam amavisd-new spamassassin spamc courier-base courier-authdaemon courier-authlib-mysql courier-imap courier-imap-ssl courier-pop courier-pop-ssl courier-ssl

During the installation of MySQL you will be prompted for the root user password, as shown:

.. image:: _static/images/mysql-step1.png

Enter a secure password, and don’t forget it!

Similarly, during the installation of Courier you will be presented with the following configuration prompts:



Choose No



Choose OK



Choose Internet Site



Enter your mail server name (e.g. replace example.com with your own domain). Make sure you have this subdomain configured in your DNS records.



Choose OK

I won’t walk you through the parameters we’re using when configuring Postfix as I want to keep this guide light. If you’re interested, you can find more information from the man pages.


.. code-block:: sh

   mv /etc/postfix/main.cf{,.default}
   vi /etc/postfix/main.cf

Copy/paste the following (change all instances of mail.example.com):

.. code-block:: ini

   myorigin = /etc/mailname
smtpd_banner = $myhostname ESMTP $mail_name
biff = no
append_dot_mydomain = no
readme_directory = no
mydestination =
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mynetworks_style = host
mailbox_size_limit = 0
virtual_mailbox_limit = 0
recipient_delimiter = +
inet_interfaces = all
message_size_limit = 0
 
# SMTP Authentication (SASL)

smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain =
 
# Encrypted transfer (SSL/TLS)
 
smtp_use_tls = yes
smtpd_use_tls = yes
smtpd_tls_cert_file = /etc/ssl/private/mail.example.com.crt
smtpd_tls_key_file = /etc/ssl/private/mail.example.com.key
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
 
# Basic SPAM prevention
 
smtpd_helo_required = yes
smtpd_delay_reject = yes
disable_vrfy_command = yes
smtpd_sender_restrictions = permit_sasl_authenticated, permit_mynetworks, reject
smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject
 
# Force incoming mail to go through Amavis
 
content_filter = amavis:[127.0.0.1]:10024
receive_override_options = no_address_mappings
 
# Virtual user mappings
 
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
virtual_mailbox_base = /var/spool/mail/virtual
virtual_mailbox_maps = mysql:/etc/postfix/maps/user.cf
virtual_uid_maps = static:5000
virtual_gid_maps =  static:5000
virtual_alias_maps = mysql:/etc/postfix/maps/alias.cf
virtual_mailbox_domains = mysql:/etc/postfix/maps/domain.cf

.. code-block:: sh

   mv /etc/postfix/master.cf{,.default}
   vi /etc/postfix/master.cf

Copy/paste the following (no changes required):

.. code-block:: ini

   
#
#
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master").
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (yes)   (never) (100)
# ==========================================================================
smtp      inet  n       -       -       -       -       smtpd
smtps     inet  n       -       -       -       -       smtpd
  -o smtpd_tls_wrappermode=yes
submission inet n       -       -       -       -       smtpd
pickup    fifo  n       -       -       60      1       pickup
  -o content_filter=
  -o receive_override_options=no_header_body_checks
cleanup   unix  n       -       -       -       0       cleanup
qmgr      fifo  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       -       1000?   1       tlsmgr
rewrite   unix  -       -       -       -       -       trivial-rewrite
bounce    unix  -       -       -       -       0       bounce
defer     unix  -       -       -       -       0       bounce
trace     unix  -       -       -       -       0       bounce
verify    unix  -       -       -       -       1       verify
flush     unix  n       -       -       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       -       -       -       smtp
# When relaying mail as backup MX, disable fallback_relay to avoid MX loops
relay     unix  -       -       -       -       -       smtp
	-o smtp_fallback_relay=
showq     unix  n       -       -       -       -       showq
error     unix  -       -       -       -       -       error
retry     unix  -       -       -       -       -       error
discard   unix  -       -       -       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       -       -       -       lmtp
anvil     unix  -       -       -       -       1       anvil
scache    unix  -       -       -       -       1       scache
#
# ====================================================================
# Interfaces to non-Postfix software. Be sure to examine the manual
# pages of the non-Postfix software to find out what options it wants.
#
# Many of the following services use the Postfix pipe(8) delivery
# agent.  See the pipe(8) man page for information about ${recipient}
# and other message envelope options.
# ====================================================================
#
# maildrop. See the Postfix MAILDROP_README file for details.
# Also specify in main.cf: maildrop_destination_recipient_limit=1
#
maildrop  unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
#
# See the Postfix UUCP_README file for configuration details.
#
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix	-	n	n	-	2	pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
  ${nexthop} ${user}
amavis    unix -        -       -       -       2       smtp
  -o smtp_data_done_timeout=1200
  -o smtp_send_xforward_command=yes
  -o disable_dns_lookups=yes
  -o max_use=20
127.0.0.1:10025 inet n  -       -       -       -       smtpd
  -o content_filter=
  -o local_recipient_maps=
  -o relay_recipient_maps=
  -o smtpd_restriction_classes=
  -o smtpd_delay_reject=no
  -o smtpd_client_restrictions=permit_mynetworks,reject
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_mynetworks,reject
  -o smtpd_data_restrictions=reject_unauth_pipelining
  -o smtpd_end_of_data_restrictions=
  -o mynetworks=127.0.0.0/8
  -o smtpd_error_sleep_time=0
  -o smtpd_soft_error_limit=1001
  -o smtpd_hard_error_limit=1000
  -o smtpd_client_connection_count_limit=0
  -o smtpd_client_connection_rate_limit=0
  -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks
As all our mail users are going to be virtual (i.e. we’re not going to create physical user accounts for each user), we only need to create one mail directory and one user account.

groupadd virtual -g 5000
useradd -r -g "virtual" -G "users" -c "Virtual User" -u 5000 virtual
mkdir /var/spool/mail/virtual
chown virtual:virtual /var/spool/mail/virtual
Make sure that, if the UID or GID differs from 5000, you update the virtual_uid_maps and virtual_gid_maps values in /etc/postfix/main.cf, and MYSQL_UID_FIELD and MYSQL_GID_FIELD in /etc/courier/authmysqlrc (later in this guide).

Now we’ll create the database which will store the mail user configuration and forwarding rules.

.. code-block:: sh

   mysql -uroot -p
   
Enter the password you created during the MySQL installation.

Copy/paste the following (change mailuserpassword, example.com and change the admin’s password to something more secure):

.. code-block:: mysql

   CREATE DATABASE mail;
   
GRANT ALL ON mail.* TO mail@localhost IDENTIFIED BY 'mailuserpassword';

.. code-block:: mysql

   FLUSH PRIVILEGES;
   USE mail;
 
CREATE TABLE IF NOT EXISTS `alias` (
  `source` varchar(255) NOT NULL,
  `destination` varchar(255) NOT NULL default '',
  `enabled` tinyint(1) unsigned NOT NULL default '1',
  PRIMARY KEY  (`source`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
 
CREATE TABLE IF NOT EXISTS `domain` (
  `domain` varchar(255) NOT NULL default '',
  `transport` varchar(255) NOT NULL default 'virtual:',
  `enabled` tinyint(1) unsigned NOT NULL default '1',
  PRIMARY KEY  (`domain`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
 
CREATE TABLE IF NOT EXISTS `user` (
  `email` varchar(255) NOT NULL default '',
  `password` varchar(255) NOT NULL default '',
  `name` varchar(255) default '',
  `quota` varchar(255) default NULL,
  `enabled` tinyint(1) unsigned NOT NULL default '1',
  PRIMARY KEY  (`email`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
 
INSERT INTO `alias` (`source`, `destination`, `enabled`) VALUES ('@localhost', 'admin@example.com', 1);
INSERT INTO `alias` (`source`, `destination`, `enabled`) VALUES ('@localhost.localdomain', '@localhost', 1);
INSERT INTO `domain` (`domain`, `transport`, `enabled`) VALUES ('localhost', 'virtual:', 1);
INSERT INTO `domain` (`domain`, `transport`, `enabled`) VALUES ('localhost.localdomain', 'virtual:', 1);
INSERT INTO `domain` (`domain`, `transport`, `enabled`) VALUES ('example.com', 'virtual:', 1);
INSERT INTO `user` (`email`, `password`, `name`, `quota`, `enabled`) VALUES ('admin@example.com', ENCRYPT('changeme'), 'Administrator', NULL, 1);
Note that we’re encrypting the password. Some guides will recommend storing the password in clear text so that you can configure Postfix to support CRAM-* (e.g. CRAM-MD5) authentication methods. I think it’s much more secure to store these passwords encrypted and use SSL/TLS to encrypt your authentication requests. For that reason, we don’t need to store clear text passwords and we don’t need to provide CRAM-* support.

Now that the database is in place we can create the map files to tell Postfix how to communicate with it.

.. code-block:: sh

   mkdir /etc/postfix/maps
   vi /etc/postfix/maps/alias.cf

Copy/paste the following (change mailuserpassword):

.. code-block:: ini

   user=mail
password=mailuserpassword
dbname=mail
table=alias
select_field=destination
where_field=source
hosts=127.0.0.1
additional_conditions=and enabled = 1

.. code-block:: sh

   vi /etc/postfix/maps/domain.cf

Copy/paste the following (change mailuserpassword):

.. code-block:: sh

   user = mail
password = mailuserpassword
dbname = mail
table = domain
select_field = domain
where_field = domain
hosts = 127.0.0.1
additional_conditions = and enabled = 1

.. code-block:: sh

   vi /etc/postfix/maps/user.cf

Copy/paste the following (change mailuserpassword):

.. code-block:: sh

   user = mail
password = mailuserpassword
dbname = mail
table = user
select_field = CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/')
where_field = email
hosts = 127.0.0.1
additional_conditions = and enabled = 1
Set restrictive read permissions as these files contain the MySQL mail user’s password.

.. code-block:: sh

   chmod 700 /etc/postfix/maps/*
   
chown postfix:postfix /etc/postfix/maps/*
The final part of configuring Postfix is to configure the authentication mechanism. SASL is a authentication layer that provides the ability to receive a user’s credentials in a variety of formats.

.. code-block:: sh

   mkdir -p /var/spool/postfix/var/run/saslauthd
   mkdir /etc/postfix/sasl
   adduser postfix sasl
   vi /etc/postfix/sasl/smtpd.conf

Copy/paste the following (change mailuserpassword):

pwcheck_method: saslauthd
auxprop_plugin: sql
mech_list: plain login
sql_engine: mysql
sql_hostnames: 127.0.0.1
sql_user: mail
sql_passwd: mailuserpassword
sql_database: mail
sql_select: SELECT password FROM user WHERE email='%u@%r' AND enabled = 1

.. code-block:: sh

   chmod -R 700 /etc/postfix/sasl/smtpd.conf
   mv /etc/default/saslauthd{,.default}
   vi /etc/default/saslauthd

Copy/paste the following (no changes required):

.. code-block:: sh

   START=yes
DESC="SASL Authentication Daemon"
NAME="saslauthd"
MECHANISMS="pam"
MECH_OPTIONS=""
THREADS=5
OPTIONS="-r -c -m /var/spool/postfix/var/run/saslauthd"

.. code-block:: sh

   vi /etc/pam.d/smtp

Copy/paste the following (change all instances of mailuserpassword):

.. code-block:: sh

   
auth    required   pam_mysql.so user=mail passwd=mailuserpassword host=127.0.0.1 db=mail table=user usercolumn=email passwdcolumn=password crypt=1
account sufficient pam_mysql.so user=mail passwd=mailuserpassword host=127.0.0.1 db=mail table=user usercolumn=email passwdcolumn=password crypt=1

.. code-block:: sh

   chmod 700 /etc/pam.d/smtp

Now let’s configure Courier.

I like to provide both IMAP and POP3 support, although personally I only use IMAP. In addition, we’ll be provide SSL support for securing authentication requests.

.. code-block:: sh

   mv /etc/courier/authdaemonrc{,.default}
   vi /etc/courier/authdaemonrc

Copy/paste the following (no changes required):

.. code-block:: sh

   authmodulelist="authmysql"
authmodulelistorig="authuserdb authpam authpgsql authldap authmysql authcustom authpipe"
daemons=5
authdaemonvar=/var/run/courier/authdaemon
DEBUG_LOGIN=0
DEFAULTOPTIONS=""
LOGGEROPTS=""

.. code-block:: sh

   mv /etc/courier/authmysqlrc{,.default}
   vi /etc/courier/authmysqlrc

Copy/paste the following (change mailuserpassword):

.. code-block:: sh

   MYSQL_SERVER localhost
MYSQL_USERNAME mail
MYSQL_PASSWORD mailuserpassword
MYSQL_PORT 0
MYSQL_DATABASE mail
MYSQL_USER_TABLE user
MYSQL_CRYPT_PWFIELD password
MYSQL_UID_FIELD 5000
MYSQL_GID_FIELD 5000
MYSQL_LOGIN_FIELD email
MYSQL_HOME_FIELD "/var/spool/mail/virtual"
MYSQL_MAILDIR_FIELD CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/')
MYSQL_NAME_FIELD name
MYSQL_QUOTA_FIELD quota

.. code-block:: sh

   mv /etc/courier/imapd{,.default}
   vi /etc/courier/imapd

Copy/paste the following (no changes required):

.. code-block:: sh

   ADDRESS=0
PORT=143
MAXDAEMONS=40
MAXPERIP=20
PIDFILE=/var/run/courier/imapd.pid
TCPDOPTS="-nodnslookup -noidentlookup"
LOGGEROPTS="-name=imapd"
IMAP_CAPABILITY="IMAP4rev1 UIDPLUS CHILDREN NAMESPACE THREAD=ORDEREDSUBJECT THREAD=REFERENCES SORT QUOTA IDLE"
IMAP_KEYWORDS=1
IMAP_ACL=1
IMAP_CAPABILITY_ORIG="IMAP4rev1 UIDPLUS CHILDREN NAMESPACE THREAD=ORDEREDSUBJECT THREAD=REFERENCES SORT QUOTA AUTH=CRAM-MD5 AUTH=CRAM-SHA1 AUTH=CRAM-SHA256 IDLE"
IMAP_PROXY=0
IMAP_PROXY_FOREIGN=0
IMAP_IDLE_TIMEOUT=60
IMAP_MAILBOX_SANITY_CHECK=1
IMAP_CAPABILITY_TLS="$IMAP_CAPABILITY AUTH=PLAIN"
IMAP_CAPABILITY_TLS_ORIG="$IMAP_CAPABILITY_ORIG AUTH=PLAIN"
IMAP_DISABLETHREADSORT=0
IMAP_CHECK_ALL_FOLDERS=0
IMAP_OBSOLETE_CLIENT=0
IMAP_UMASK=022
IMAP_ULIMITD=65536
IMAP_USELOCKS=1
IMAP_SHAREDINDEXFILE=/etc/courier/shared/index
IMAP_ENHANCEDIDLE=0
IMAP_TRASHFOLDERNAME=Trash
IMAP_EMPTYTRASH=Trash:7
IMAP_MOVE_EXPUNGE_TO_TRASH=0
SENDMAIL=/usr/sbin/sendmail
HEADERFROM=X-IMAP-Sender
IMAPDSTART=YES
MAILDIRPATH=Maildir

.. code-block:: sh

   mv /etc/courier/imapd-ssl{,.default}
   vi /etc/courier/imapd-ssl

Copy/paste the following (change mail.example.com):

.. code-block:: sh

   SSLPORT=993
SSLADDRESS=0
SSLPIDFILE=/var/run/courier/imapd-ssl.pid
SSLLOGGEROPTS="-name=imapd-ssl"
IMAPDSSLSTART=YES
IMAPDSTARTTLS=YES
IMAP_TLS_REQUIRED=0
COURIERTLS=/usr/bin/couriertls
TLS_KX_LIST=ALL
TLS_COMPRESSION=ALL
TLS_CERTS=X509
TLS_CERTFILE=/etc/ssl/private/mail.example.com.pem
TLS_TRUSTCERTS=/etc/ssl/certs
TLS_VERIFYPEER=NONE
TLS_CACHEFILE=/var/lib/courier/couriersslcache
TLS_CACHESIZE=524288
MAILDIRPATH=Maildir

.. code-block:: sh

   mv /etc/courier/pop3d{,.default}
vi /etc/courier/pop3d
Copy/paste the following (no changes required):

.. code-block:: sh

   PIDFILE=/var/run/courier/pop3d.pid
MAXDAEMONS=40
MAXPERIP=4
POP3AUTH="LOGIN"
POP3AUTH_ORIG="PLAIN LOGIN CRAM-MD5 CRAM-SHA1 CRAM-SHA256"
POP3AUTH_TLS="LOGIN PLAIN"
POP3AUTH_TLS_ORIG="LOGIN PLAIN"
POP3_PROXY=0
PORT=110
ADDRESS=0
TCPDOPTS="-nodnslookup -noidentlookup"
LOGGEROPTS="-name=pop3d"
POP3DSTART=YES
MAILDIRPATH=Maildir
mv /etc/courier/pop3d-ssl{,.default}
vi /etc/courier/pop3d-ssl
Copy/paste the following (change mail.example.com):

SSLPORT=995
SSLADDRESS=0
SSLPIDFILE=/var/run/courier/pop3d-ssl.pid
SSLLOGGEROPTS="-name=pop3d-ssl"
POP3DSSLSTART=YES
POP3_STARTTLS=YES
POP3_TLS_REQUIRED=0
COURIERTLS=/usr/bin/couriertls
TLS_STARTTLS_PROTOCOL=TLS1
TLS_KX_LIST=ALL
TLS_COMPRESSION=ALL
TLS_CERTS=X509
TLS_CERTFILE=/etc/ssl/private/mail.example.com.pem
TLS_TRUSTCERTS=/etc/ssl/certs
TLS_VERIFYPEER=NONE
TLS_CACHEFILE=/var/lib/courier/couriersslcache
TLS_CACHESIZE=524288
MAILDIRPATH=Maildir
We need to create SSL certificates for Courier to use when authenticating using SSL/TLS. You can either purchase these (to prevent “invalid” certificate warnings) or generate a self-signed certificate which is just as secure, and free.

Run the following (change mail.example.com):

.. code-block:: sh

   
# Remove default certificates
rm -f /etc/courier/imapd.cnf
rm -f /etc/courier/imapd.pem
rm -f /etc/courier/pop3d.cnf
rm -f /etc/courier/pop3d.pem
 
# Generate a new PEM certificate (valid for 10 years)
openssl req -x509 -newkey rsa:1024 -keyout "/etc/ssl/private/mail.example.com.pem" -out "/etc/ssl/private/mail.example.com.pem" -nodes -days 3650
 
# Generate a new CRT certificate (valid for 10 years)
openssl req -new -outform PEM -out "/etc/ssl/private/mail.example.com.crt" -newkey rsa:2048 -nodes -keyout "/etc/ssl/private/mail.example.com.key" -keyform PEM -days 3650 -x509

.. code-block:: sh

   
chmod 640 /etc/ssl/private/mail.example.com.*
chgrp ssl-cert /etc/ssl/private/mail.example.com.*
You will be prompted to input some information about the certificates you create. You can enter any information you want here except Common Name (CN) which must be your mailname (e.g. mail.example.com).

Next we’ll configure Amavis, the software that ties together SpamAssassin and ClamAV with Postfix.

.. code-block:: sh

   
adduser clamav amavis
cat /dev/null > /etc/amavis/conf.d/15-content-filter-mode
vi /etc/amavis/conf.d/15-content-filter-mode
Copy/paste the following (no changes required):

use strict;
 
.. code-block:: sh

   
@bypass_virus_checks_maps = (
   \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
 
@bypass_spam_checks_maps = (
   \%bypass_spam_checks, \@bypass_spam_checks_acl, \$bypass_spam_checks_re);
 
1;

.. code-block:: sh

   cat /dev/null > /etc/amavis/conf.d/50-user
vi /etc/amavis/conf.d/50-user
Copy/paste the following (no changes required):

use strict;
 
.. code-block:: sh

   
@local_domains_acl = qw(.);
$log_level = 1;
$syslog_priority = 'info';
$sa_kill_level_deflt = 6.5;
$final_spam_destiny = D_DISCARD;
$pax = 'pax';
 
1;

.. code-block:: sh

   mv /etc/default/spamassassin{,.default}
vi /etc/default/spamassassin
Copy/paste the following (no changes required):

ENABLED=1
OPTIONS="--create-prefs --max-children 5 --helper-home-dir"
PIDFILE="/var/run/spamd.pid"
CRON=0
dpkg-reconfigure clamav-freshclam


Choose OK



Choose daemon



Choose a mirror closest to you.



Enter your proxy, if required. Usually you will leave this blank.



By default, ClamAV updates every hour. That’s excessive. Bring that down to once a day.



Choose No

Now restart everything.

.. code-block:: sh

   
/etc/init.d/saslauthd restart
/etc/init.d/postfix restart
/etc/init.d/courier-authdaemon restart
/etc/init.d/courier-imap restart
/etc/init.d/courier-imap-ssl restart
That’s it, you’re done!

You can test your setup by configuring your mail client to connect to your new mail server using admin@example.com as your username and the password you chose (“changeme” in the guide).

Errors will usually show up in /var/log/mail.log or post any problems you’re having in a comment and I’ll try my best to help.

For more information regarding the mail database, testing using Telnet, and more information regarding how all these services are stitched together, please see Flurdy’s guide.
