## Install Spamassassin filter for Postfix
SpamAssassin is an intelligent email filter which uses a diverse range of tests to identify unsolicited bulk email, more commonly known as "spam". These tests are applied to email headers and content to classify email using advanced statistical methods. In addition, SpamAssassin has a modular architecture that allows other technologies to be quickly wielded against spam and is designed for easy integration into virtually any email system.

On the shell, as `root` and `vim` as our editor
### Install Spamassassin
~~~~
apt-get install spamassassin spamc
~~~~

### Create a user and group for spamd service:
~~~~
groupadd spamd
useradd -g spamd -s /bin/false -d /var/log/spamassassin spamd
mkdir /var/log/spamassassin
chown spamd:spamd /var/log/spamassassin
~~~~

### Configuring Spamassassin
Edit /etc/default/spamassassin so it matches this block:
~~~~
# Change to "1" to enable spamd on systems using sysvinit:
ENABLED=1

# Options
# See man spamd for possible options. The -d option is automatically added.
# SpamAssassin uses a preforking model, so be careful! You need to
# make sure --max-children is not set to anything higher than 5,
# unless you know what you're doing.

OPTIONS="--create-prefs --max-children 5 --helper-home-dir --username spamd \
-H /var/log/spamassassin/ -s /var/log/spamassassin/spamd.log"

# Cronjob
# Set to anything but 0 to enable the cron job to automatically update
# spamassassin's rules on a nightly basis
CRON=1
~~~~

### Start Spamassassin
~~~~
service spamassassin start
~~~~
This block demostrates spam daemon running on port **`783`**
~~~~
netstat -tulpn | grep 783
tcp        0      0 127.0.0.1:783           0.0.0.0:*               LISTEN      10803/perl
tcp6       0      0 ::1:783                 :::*                    LISTEN      10803/perl
~~~~

### Configure Postfix to use Spamassassin as a filter
~~~~
vim /etc/postfix/master.cf
~~~~
This line here:
~~~~
smtp      inet  n       -       y       -       -       smtpd
~~~~
Make it look like this:
~~~~
smtp      inet  n       -       y       -       -       smtpd -o content_filter=spamassassin
~~~~

Place this at the bottom of the file, this is one line:
~~~~
spamassassin unix -     n       n       -       -       pipe
        user=spamd argv=/usr/bin/spamc -f -e  
        /usr/sbin/sendmail -oi -f ${sender} ${recipient}
~~~~
Now, save the file, time to restart **`postfix`**:
~~~~
service postfix restart
~~~~

### Edit the spamassassin config
~~~~
vim /etc/spamassassin/local.cf
~~~~
Modify the Subject lines of emails ID'd as spam
~~~~
###########################################################################
#   Add *****SPAM***** to the Subject header of spam e-mails
rewrite_header Subject [***** SPAM _SCORE_ *****]
~~~~
~~~~
#   Set the threshold at which a message is considered spam (default: 5.0)
#
required_score 5.0
~~~~

Enable bayes, bayes auto learn by uncommenting or simply add these two lines:
~~~~
#   Use Bayesian classifier (default: 1)
#
use_bayes 1
~~~~
~~~~
#   Bayesian classifier auto-learning (default: 1)
#
bayes_auto_learn 1
~~~~

And then restart spamassassin & postfix:
~~~~
service spamassassin restart
~~~~
~~~~
service postfix restart
~~~~
### Make sure spamassassin is enabled to start upon bootup.
~~~~
systemctl enable spamassassin.service
~~~~

### So, here's a view of the **`spamd.log`**
~~~~
vim /var/log/spamassassin/spamd.log
~~~~

~~~~
Fri Feb  1 05:22:11 2019 [9674] info: spamd: server started on IO::Socket::IP [::1]:783, IO::Socket::IP [127.0.0.1]:783 (running version 3.4.2)
Fri Feb  1 05:22:11 2019 [9674] info: spamd: server pid: 9674
Fri Feb  1 05:22:11 2019 [9674] info: spamd: server successfully spawned child process, pid 9675
Fri Feb  1 05:22:11 2019 [9674] info: spamd: server successfully spawned child process, pid 9676
Fri Feb  1 05:22:11 2019 [9674] info: prefork: child states: II
Fri Feb  1 05:22:37 2019 [9675] info: spamd: connection from ::1 [::1]:56440 to port 783, fd 5
Fri Feb  1 05:22:37 2019 [9675] info: spamd: creating default_prefs: /var/log/spamassassin/.spamassassin/user_prefs
Fri Feb  1 05:22:37 2019 [9675] warn: config: created user preferences file: /var/log/spamassassin/.spamassassin/user_prefs
Fri Feb  1 05:22:37 2019 [9675] info: spamd: processing message <CAFbqaXj1oBWKyVG34FtQ5w6+X1v9C-BZa5jPaH=3FTwwXtbUcghw@mail.gmail.com> for spamd:1002
Fri Feb  1 05:22:38 2019 [9675] info: spamd: identified spam (1000.0/5.0) for spamd:1002 in 0.3 seconds, 3114 bytes.
Fri Feb  1 05:22:38 2019 [9675] info: spamd: result: Y 1000 - FREEMAIL_FROM,GTUBE,HTML_MESSAGE,RCVD_IN_DNSWL_NONE,SPF_PASS,TVD_SPACE_RATIO,T_DKIM_INVALID scantime=0.3,size=3114,user=spamd,uid=1002,required_score=5.0,rhost=::1,raddr=::1,rport=56440,mid=<CAFbqaXj1oBa3bvexeFtQ5w6+X1v9C-BZa5jPaH=3FJttbUcghw@mail.gmail.com>,autolearn=no autolearn_force=no
~~~~

I sent a bogus email from an outside source towards the server:
~~~~
Fri Feb  1 05:22:38 2019 [9675] info: spamd: identified spam (1000.0/5.0) for spamd:1002 in 0.3 seconds, 3114 bytes.
~~~~

Here is the sample text you should send from outside your network: [The GTUBE](https://spamassassin.apache.org/gtube//)
~~~~
XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X
~~~~
**` Note that this should be reproduced in one line, without whitespace or line breaks. `**


### Sample source from email received: **`SPAM`**
~~~~
Received: from localhost by mail.yourdomain.com
	with SpamAssassin (version 3.4.2);
	Fri, 01 Feb 2019 05:22:38 +0000
From: YourName T <yourname@gmail.com>
To: Your Name <yourname@yourdomain.com>
Subject: [***** SPAM 1000.0 *****] 
Date: Thu, 31 Jan 2019 21:22:24 -0800
Message-Id: <CAFbqaDk3oBWKyZC90FtQ5w6+X1v9C-BZ5jSwaH=3FJttbUcghw@mail.gmail.com>
X-Spam-Checker-Version: SpamAssassin 3.4.2 (2018-09-13) on mail.yourdomain.com
X-Spam-Flag: YES
X-Spam-Level: **************************************************
X-Spam-Status: Yes, score=1000.0 required=5.0 tests=FREEMAIL_FROM,GTUBE,
	HTML_MESSAGE,RCVD_IN_DNSWL_NONE,SPF_PASS,TVD_SPACE_RATIO,
	T_DKIM_INVALID autolearn=no autolearn_force=no version=3.4.2
MIME-Version: 1.0
~~~~

### Sample source from email received: **`NOT SPAM`**
~~~~
Received: by mail.yourdomain.com (Postfix, from userid 1002)
	id 2BC33404A3; Fri,  1 Feb 2019 05:46:57 +0000 (UTC)
Authentication-Results: mail.yourdomain.com;
	dkim=pass (2048-bit key; unprotected) header.d=gmail.com header.i=@gmail.com header.b=HtEwRGZq;
	dkim-atps=neutral
X-Spam-Checker-Version: SpamAssassin 3.4.2 (2018-09-13) on mail.yourdomain.com
X-Spam-Level: 
X-Spam-Status: No, score=0.0 required=5.0 tests=FREEMAIL_FROM,HTML_MESSAGE,
	RCVD_IN_DNSWL_NONE,RCVD_IN_MSPIKE_H2,SPF_PASS,T_DKIM_INVALID
	autolearn=ham autolearn_force=no version=3.4.2
~~~~

### Spamassassin being applied to email
~~~~
Feb 14 21:57:49 mail postfix/submission/pipe[12872]: 1181A3F5FC: to=<youruser@yourdomain.com>, relay=spamassassin, delay=1.5, delays=1.2/0.01/0/0.32, dsn=2.0.0, status=sent (delivered via spamassassin service)
~~~~

### Read more about this topic:
[https://wiki.apache.org/spamassassin/UsingPyzor](https://wiki.apache.org/spamassassin/UsingPyzor)<br>
[https://www.binarytides.com/install-spamassassin-with-postfix-dovecot/](https://www.binarytides.com/install-spamassassin-with-postfix-dovecot/)

This completes the Postfix && SpamAssassin setup.
<br>
### Next up: [Sieve Filtering](https://github.com/insideTheFlux/Mail-Server-with-Extras/blob/insideTheFlux-Ubuntu_18-04/SieveFiltering.md)
