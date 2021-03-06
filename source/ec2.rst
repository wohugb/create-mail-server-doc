Elastic Compute Cloud
============================
Impressions

Easy to use. Anyone can use, not just big companies. Very useful. Tools are command line but simple. Firefox extensions work well. Recommended.

I find it very usefull. Basically it is a colo hosting environment. Some may use it as for Saas, ie single scalable application in the cloud, but I use it as a hosting environment for complete servers.

ec2 introduction, tips and howtos

I have made a separate tips and howto on the use of ec2 for general server needs. Hope it will be useful for people.

How I use it for my mail servers

Different images to launch for different needs. Good way to scale backup MXs if needed. Can script backup to S3 of mail dirs etc.

Using EC2 with this howto

If you plan to use EC2 to follow this howto, then familiarise yourself with EC2 first. Check the links further down, e.g. my tips.

Once competent enough on EC2, launch the latest official Ubuntu ec2 image or one of Eric Hammond images. You can cheat by using my other images, but you should really know how the whole server was built by starting from the bottom.

When using EC2 images, be aware of security groups as they restricts access to your server on top of the firewall. Initially you will need SSH (22) access, quite soon you will need SMTP and IMAP ports opened, 25,143,465,587 and 993, and eventually webserver ports of 80 and 443. Read here for tips on securing AMIs.

Also do not terminate your instances without backing up your machine. This you can do by either create your own image. Or backup certain data if you got an image to instantiate from. Back up to S3 or your local machine. Create images only now and then. Backup configurations, database, maildirs more regularily.

Once launched, follow my Initialize section.

1st note: Spamhaus.org lists amazons ec2 ip ranges as dynamic, thus many mail servers will reject emails from it. (Including other people using this howto.) But Spamhaus has a simple web page to remove ips, which they link to in rejection messages. Simple look in your logs, click on the link on follow the instrucions: basically fill in your ip, email and state its for a mail server. Then Spamhaus will remove your IP from their database.

2nd Note: Amazon has extended this spam limitations, so if you have a busy mail server, follow their FAQ entry for removing mail throttling.

3rd Note: This fix needs to applied to the instances buildt on an early 8.04 base. This is not a problem with the later Hammond or any Canonical based images.

4th Note: Check I have not left my SSH key in the root or ubuntu user.

5th Note: Please let me know if I've been silly enough to leave a bash history or anything like that?

Amazon EC2 Images: AMIs

Public AMIs to use as base:

Current AMIs:

AMI	Description	S3 Name	Ubuntu
version	Extended from	Status
US E:
ami-2d4aa444
Ubuntu's official server AMI
ubuntu-images/ubuntu-lucid-10.04-i386-server-20100427.1
10.04 LTS Lucid
OK
US W:
ami-c997c68c
EU:
ami-cf4d67bb
Asia:
ami-57f28d05
US E:
ami-f8a64e91
ami-5833c831
My base Ubuntu server AMI. Postfix not configured.
flurdy-amis/ubuntu-lucid-base-20100604-02
10.04 LTS Lucid
ami-2d4aa444
Ubuntu's official base
OK
EU:
ami-91341ee5
ami-cf4d67bb
US E:
ami-9c947cf5
ami-a23ec5cb
Basic Postfix server, with MySql, amavis, TLS.
flurdy-amis/ubuntu_lucid_mailserver_base-20110701-01
10.04 LTS Lucid
ami-f8a64e91
flurdy's base
Testing
US E
ami-0a957d63
ami-f2758c9b
With Courier IMAP, SASL
flurdy-amis/ubuntu-lucid-mail-imap-20100610-01
10.04 LTS Lucid
ami-9c947cf5
Basic Postfix
Testing
US E
ami-c0ee06a9
ami-f0758c99
With RoundCube Webmail, PhpMyAdmin
flurdy-amis/ubuntu-lucid-mail-web-20100611-01
10.04 LTS Lucid
ami-0a957d63
With Courier
Testing


Older AMIs:

( Images with strike through them are no longer recommended. Their are fine for experimenting and testing, but should not be used for permanent "live" servers )
AMI	Description	S3 Name	Ubuntu
version	Extended from
ami-1515f67c
Base install: Canonical's Official Ubuntu 9.10 Karmic 32bit US
canonical-cloud-us/ubuntu-karmic-9.10-i386-server-20091027.1
9.10 Karmic
ami-5059be39
Base install: Canonical's Official Ubuntu 8.10 Intrepid 32bit US
canonical-cloud-us/ubuntu-intrepid-20090422-i386
8.10 Intrepid
ami-c4f615ad
Base install: Alestic Ubuntu 8.04 LTS Hardy 32bit US
alestic/ubuntu-8.04-hardy-base-20091011
8.04 LTS Hardy
ami-ce44a1a7
Base install: Alestic Ubuntu 8.04 LTS Hardy 32bit US
alestic/ubuntu-8.04-hardy-base-20080430
8.04 LTS Hardy
ami-4132d428
Clean with all packages installed but no configuration
flurdy-amis/ubuntu-mail-server-clean-20090529-2
8.10 Intrepid
ami-5059be39 (Canonical)
ami-0f41a466
Clean with all packages installed but no configuration
flurdy-amis/ubuntu-mail-server-clean-080502-1
8.04 LTS Hardy
ami-ce44a1a7 (Old Alestic)
ami-eb39df82
Just mysql, postfix and courier configured
flurdy-amis/ubuntu-mail-server-basic-20090604-1
8.10 Intrepid
ami-4132d428 (Clean Canonical)
ami-8541a4ec
Just mysql, postfix and courier configured
flurdy-amis/ubuntu-mail-server-simple-080504-1
8.04 LTS Hardy
ami-0f41a466 (Clean Alestic)
ami-9941a4f0
Including anti spam and anti virus
flurdy-amis/ubuntu-mail-server-spam-080504-1
8.04 LTS Hardy
ami-8541a4ec (Simple)
ami-395fba50
Including TLS and SASL encryption and authentication
flurdy-amis/ubuntu-mail-server-secure-080527-2
8.04 LTS Hardy
ami-9941a4f0 (Spam)
ami-275fba4e
With webmail and admin enabled
flurdy-amis/ubuntu-mail-server-webmail-080527-1
8.04 LTS Hardy
ami-395fba50 (Secure)
ami-xxx
With back up mx
flurdy-amis/ubuntu-mail-server-backup-xxx
8.04 LTS Hardy
ami-275fba4e (Webmail)
ami-xxx
With back up mx only
flurdy-amis/ubuntu-mail-server-backup-only-xxx
8.04 LTS Hardy
ami-395fba50 (Secure)
If you have a comment or question about the ec2 images, please discuss it in the forums?
If you notice a security problem, or I have not cleaned the images properly please let me know?
EC2 Links
Amazon web services (AWS)
Elastic Computing Cloud (EC2)
Simple Storage Service (S3)
AWS Cost Calculator
EC2 Resource Centre
EC2 Starter Guide
EC2 Firefox extension: Elasticfox
Elasticfox for Firefox 3
S3 Firefox extension: S3Fox
EC2 to S3 Admin Scripts
Eric Hammond 8.04 AMI
alestic, ubuntu ec2 images
Ubuntu ec2
Ubuntu Cloud
Ubuntu ec2 Starter Guide
My ec2 Tips and Howtos
