测试
=====

常见问题
----------------------

* Missed a step

 If you mistakenly or intentially skipped past sections, you may have missed an important step in your configuration, which my guide pressumes you have followed.

* Typo

 99% of all problems is spelling errors or typos you entered while following this guide. Sorry, but it just happens. Often it can be trivial, such as a space at the end of the configuration line which was not expected etc. Or not understanding my example where it is a multi line entry.

* Typo by me

 Yes, I make a lot of mistakes. Nothing wrong in that, but I hope I have corrected most over time. Any new sections are however at risk... :)

* Different application or configurations

 It is obviously entirely up to you how you set up your system. But the more you deviate from this guide, the more likely incompatibilties or confusion will arrise.

* Distrobution/version differences

 If you run a different version or even distrobution to this guide, then some things will be different. Small issues, such as default values and significant things such as path differences etc. Some sections in this guide are not always thouroughly tested with every new release of Ubuntu, but these differences gets pointed out by people for me.

* Walking before crawling
 Don't try the full blown mail server before the basics are working.

* Gamma rays and little goblins
 Got to blame it on something or someone.

测试strategy
------------------

What steps to think of when testing.

及早并经常进行测试
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is very helpfull to test early in this set up, to establish if the first sections are working as expected.

So when you only have your very basic postfix and mysql up: Test it!

That way you know that bit worked and can rule it out of any future problems.

Don't wait till you complicated and mudded the water after amavis, courier etc is added.

By constantly testing if you can send and receive you can tick off and black box each section as working, and immidietly spot issues.

隔离问题
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Testing how things work is often about isolating the problem first. So by using the steps of testing early above, you can see which step caused the problem.

Also if you can't log into your webmail it is often nothing to do with the webmail section that is causing the problem. Often postfix itself is broken etc.

Test in order
^^^^^^^^^^^^^^^^^^^^^^^^^^^

As part of the isolating the problem rule, you most of the time test in order, and test each section thus isolating the problem. This would then quickly isolate the problem when e.g. such as above issues of reading emails via the webmail. This would be in order:

#. Access: Can I get(ssh) to the box, and is there a firewall issue?
#. Database: Is the database up, do my application reach it?
#. Postfix: Can I send email by command line, do I receive emails via telnet?
#. Content checks: Do they cause a problem?
#. Courier: Can I read emails?
#. Webmail: And last but not least does the web integration work?

简化系统
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Assisting in isolating the problem, you often have to disable options and applications. Such as turn of postgrey or content checks to make sure emails to get delivered.

Previous editions do have some more detail on how to achieve this

尾，尾和尾部
-------------------------------

Essential to monitor what actually happens, and tailing specifically the mail and mysql log.

* /var/log/system.log
* /var/log/mail.log
* /var/log/mysql/mysql.log
* /var/log/apache2/access.log

In one window:

.. code-block::

   tail -f /var/log/mail.log

And in another window:

.. code-block::sh

   tail -f /var/log/mysql/mysql.log

In a third or more do your actual configuration or testing.

关闭服务
----------------------------

previous edition 1

previous edition 2

The previous editions has detail on switching services off untill time to test them. 

It also details locking down your server from spammers untill finished testing.

开启调试
------------------

Shorewall

You can also switch on more messages for when the firewall is rejecting connections. Add info to all REJECT, BOUNCE and DROP policies.

.. code-block:: sh

   sudo vi /etc/shorewall/policy

such as:

.. code-block:: ini

   net all DROP info

MySQL

There is no point in tailing the mysql log if query debugging is not turned one. 

By default it is not. However in this guide I do switch it on, in case that was missed switch it on now:

.. code-block:: sh

   sudo vi /etc/mysql/my.cnf

Make sure this is not commented out

.. code-block:: ini

   log = /var/log/mysql/mysql.log

Courier

As mentioned in the setup , switching on debugging for Courier is easy:

.. code-block:: sh

   sudo vi /etc/courier/authdaemonrc

.. code-block:: ini

   DEBUG_LOGIN=2

Amavis

You can also debug amavis:

.. code-block:: bash

   sudo vi /etc/amavis/conf.d/50-user

And perhaps bump it up if already debugging:

.. code-block:: ini

   $log_level = 2;

Telnet是你的朋友
-------------------------

When testing a mail server, telnet is alpha & omega. You use it to simulate real mail servers to test responses by your mail server.

First you test it on the server to exclude firewall and network issues.

Then you test it from another machine to simulate an actual other mail server.

Once these are working you can use proper email clients, however 99% I just use mutt locally when I need to test if a server is working.

Can postfix receive?
---------------------------

Lets assume:
You have followed my guide up to basic configuration at least
You have entered data into the database
The services MySQL and Postfix are running.
If testing a fuller stack, then amavis, postgrey, clamav-daemon, spamassassin etc must also be running.
Try this locally on the server first, then try from another machine once it is working locally.
Lets try and send a message to xandros@example.org (replace with your own user in this setup, or use postmaster@localhost) from you@example.com (again replace with a real email address you use that is not associated with this server.)

.. code-block:: bash

   telnet localhost 25
   # Open the hand shake with ehlo and the server name you are connecting from...
   # Change mail.example.com to something valid eg your servername
   EHLO mail.example.com
   # The mail server will then dump out some details about its capabilities, e.g.
   >250-mail.flurdy.net
   >250-PIPELINING
   >....
   >....
   # then say who is the sender of this email
   MAIL FROM: <your@example.com>
   > 250 Ok
   # then say who the mail is for
   RCPT TO: <xandros@example.org>
   > 250 Ok
   # then enter the keyword data
   data
   > 354 End data with <CR><LF>.<CR><LF></LF></CR></LF></CR>
   # enter message bodyand end with a line with only a full stop.
   blah blah blah
   more blah
   .
   > 250 Ok; queued as QWKJDKASAS
   # end the connection with
   quit
   > 221 BYE

If while you were doing this you were tailing the /var/log/mail.log you would see some activities and if any errors occured. (You should probably get some complaints about missing headers as we skipped most...)

If while you were doing this you were tailing the /var/log/mysql/mysql.log as well you really should have seen some activity otherwise you have a problem.

If you see any errors (or worse no activity) in these log files, this is what you need to fix! For common problems and solutions check the previous edition.

However if no errors popped up, and the folder /var/mail/virtual/xandros now exists then your server can receive emails!

Can postfix send?
--------------------

You need to make sure you can first receive emails as above

The services MySQL and Postfix are running.

Basically you just tested that above, but we need double check if it can send out to other servers. Again we will first test locally, which should work, then remotely which introduces many possible problems.

.. code-block:: bash

   telnet localhost 25
   # Open the hand shake with ehlo and the server name you are connecting from...
   # This time it has to be the name of your server
   EHLO mail.example.org
   # The mail server will then dump out some details about its capabilities, e.g.
   >250-mail.flurdy.net
   >250-PIPELINING
   >....
   >....
   # then say who is the sender of this email, which is a local user
   MAIL FROM: <xandros@example.org>
   > 250 Ok
   # then say who the mail is for which is an external address e.g. gmail etc.
   RCPT TO: <you@example.com>
   > 250 Ok
   # then enter the keyword data
   data
   > 354 End data with <CR><LF>.<CR><LF></LF></CR></LF></CR>
   # enter message bodyand end with a line with only a full stop.
   blah blah blah
   more blah
   .
   > 250 Ok; queued as QWKJDKASAS
   # end the connection with
   quit
   > 221 BYE

We have to assume receiving works above so no need to tail mysql's logs. However if any rejection errors occured in the mail.log then you have an error.

However if no errors occured and you see in the log something like this:

.. code-block:: bash

   Dec 17 10:25:45 servername postfix/smtp[12345]: 12345678: to=<you@example.com>, relay=127.0.0.1[127.0.0.1]:10024, 
   delay=15, delays=15/0.01/0.02/0.11, dsn=2.0.0, status=sent (250 2.0.0 Ok, id=12345-09, from MTA([127.0.0.1]:10025): 
   250 2.0.0 Ok: queued as 1234567)

Then the sending emails work!

Can courier read emails
---------------------------

You need to make sure you can first receive emails as above

You need to make sure you can send emails as above

You need to make sure you have received an email and the folder /var/mail/virtual/xandros exists
The services MySQL, courier-authdaemon and courier-imap are running.

There is not too much you can test via telnet for courier. But you can check if it is up and you can connect to it.

.. code-block:: bash

   telnet 127.0.0.1 143
   Trying 127.0.0.1...
   Connected to 127.0.0.1.
   Escape character is '^]'.
   * OK [CAPABILITY IMAP4rev1 UIDPLUS CHILDREN NAMESPACE THREAD=ORDEREDSUBJECT THREAD=REFERENCES SORT
     QUOTA IDLE ACL ACL2=UNION STARTTLS LOGINDISABLED] Courier-IMAP ready. Copyright 1998-2008
     Double Precision, Inc.  See COPYING for distribution information.

The rest you would have to test via a proper email IMAP client.

Can amavis check and pass emails along?

You need to make sure you can first receive emails as above

You need to make sure you can send emails as above

You need to make sure you have received an email and the folder /var/mail/virtual/xandros exists

You can check if the service is responding:

.. code-block:: bash

   telnet 127.0.0.1 10024
   Trying 127.0.0.1...
   Connected to 127.0.0.1.
   Escape character is '^]'.
   220 [127.0.0.1] ESMTP amavisd-new service ready

Then just tail /var/log/mail.log for any problems.
