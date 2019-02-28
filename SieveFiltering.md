## Get Dovecot to move spam to another folder with Sieve filtering.
Let's use Dovecot to do some email filtering for spam. For this post, we focus on spam. Be aware this technique can be used to file emails into all sorts of folders also seen from other email providers.

On the shell, as `root` and `vim` as our editor

### Install sieve, managesieve, dovecot-lmtpd
I'm assuming if you're this deep, tinkering away, that you should've had spamassassin installed by now. I mean it will detect the spam, then dovecot-lmtp will pick up on the header details and route the email appropriately.<p>
Install:
~~~~
apt-get install dovecot-sieve dovecot-managesieved dovecot-lmptd
~~~~

### Edit /etc/dovecot/dovecot.conf
This:
~~~~
protocols = " imap"
~~~~
Change to this:
~~~~
protocols = imap lmtp
~~~~

### Protocol lmtp plugin support
After the **`namespace inbox`** {} block, add this:
~~~~
protocol lmtp {
  postmaster_address = postmaster@yourdomain.com
  mail_plugins = $mail_plugins sieve
}
~~~~
Then after **`lmtp`** block, add this:
~~~~
plugin {
   sieve_default = /var/lib/dovecot/sieve/default.sieve
   sieve_global = /var/lib/dovecot/sieve/
}
~~~~

### Bind the LMTP service to unix socket
Right after **`service auth`** {} block, add this:
~~~~
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   group = postfix
   mode = 0600
   user = postfix
  }
}
~~~~

### User doesn't exist error in **`/var/log/mail.log`**
This error shows up at the end after the whole procedure is done. Instead of waiting until the end for it appear, it is better we handled it at this point. At a later date this might be fixed through dovecot sources. Read more about it ([here](https://utcc.utoronto.ca/~cks/space/blog/sysadmin/DovecotIgnoreDomainOnAuth)) & look at the source examples ([here](https://github.com/dovecot/core/blob/master/doc/example-config/conf.d/10-auth.conf))
~~~~
Feb  9 09:42:53 mail dovecot: lmtp(8985): Connect from local
Feb  9 09:42:53 mail postfix/submission/lmtp[8984]: 2357C46AA9: to=<yourname@yourdomain.com>, relay=mail.yourdomain.com[private/dovecot-lmtp], delay=0.05, delays=0.01/0.01/0.02/0.02, dsn=5.1.1, status=bounced (host mail.yourdomain.com[private/dovecot-lmtp] said: 550 5.1.1 <yourname@yourdomain.com> User doesn't exist: yourname@yourdomain.com (in reply to RCPT TO command))
Feb  9 09:42:53 mail dovecot: lmtp(8985): Disconnect from local: Successful quit
~~~~
While in **`/etc/dovecot/dovecot.conf`** add this line after **`mail_location`**
~~~~
auth_username_format = %Ln
~~~~
Save file, exit.

## Postfix **`main.cf`** Configuration
Tell Postfix to use this socket for final delivery, non virtual user setup. In **`/etc/postfix/main.cf`** & after **`smtpd_sasl_path=private/auth`** - Add this line:
~~~~
mailbox_transport = lmtp:unix:private/dovecot-lmtp
~~~~
Save file, exit.

### Create the sieve, fill it with some rules.
The plugin implements a Sieve interpreter, which filters incoming messages using a script specified in the Sieve language ([RFC 5228](http://tools.ietf.org/html/rfc5228/)). The Sieve script is provided by the user and, using that Sieve script, the user can customize how incoming messages are handled. Messages can be delivered to specific folders, forwarded, rejected, discarded, etc.<br> Read more: [Pigeonhole Sieve Interpreter](https://wiki2.dovecot.org/Pigeonhole/Sieve), [Pigeonhole Sieve Configuration](https://wiki2.dovecot.org/Pigeonhole/Sieve/Configuration#Pigeonhole_Sieve_Configuration), [Sieve scripts samples](https://p5r.uk/blog/2011/sieve-tutorial.html)
~~~~
mkdir /var/lib/dovecot/sieve/
~~~~
vim **'default.sieve`**.
~~~~
vim /var/lib/dovecot/sieve/default.sieve
~~~~
Add this into the contents of the file:
~~~~
require "fileinto";

if header :contains "X-Spam-Flag" "YES" {
        fileinto "Spam";
}
~~~~
Save file, exit.
<br>
<br>
Make sure that the working directory is **`/var/lib/dovecot/`**
~~~~
[/var/lib/dovecot/sieve]# sievec default.sieve
~~~~
Change ownership of the sieve file to dovecot specific user:
~~~~
[/var/lib/dovecot/sieve]# chown mail:mail default.sieve
~~~~

And finally, restart Dovecot,Postfix:
~~~~
service dovecot restart 
~~~~
~~~~
service postfix restart
~~~~
<br>
<br>

Here's a log sample of it in action:
~~~~
Feb  9 09:55:24 mail postfix/submission/qmgr[9282]: 0F05D3F602: removed
Feb  9 09:55:24 mail postfix/submission/cleanup[9310]: 89E8946AAB: message-id=<05a63b9034f193df12dffe1a9f79bcc2@yourdomain.com>
Feb  9 09:55:24 mail opendkim[1488]: 89E8946AAB: no signing table match for 'yourname@yourdomain.com'
Feb  9 09:55:24 mail opendkim[1488]: 89E8946AAB: no signature data
Feb  9 09:55:24 mail postfix/submission/qmgr[9282]: 89E8946AAB: from=<yourname@yourdomain.com>, size=4445, nrcpt=1 (queue active)
Feb  9 09:55:24 mail dovecot: lmtp(9317): Connect from local
Feb  9 09:55:24 mail dovecot: lmtp(yourname): 7gyQI4yjXlxlJAAAUbDMYQ: sieve: msgid=<05a63b9034f193df12dffe1a9f79bcc2@yourdomain.com>: stored mail into mailbox 'Spam'
Feb  9 09:55:24 mail postfix/submission/lmtp[9316]: 89E8946AAB: to=<yourname@yourdomain.com>, relay=mail.yourdomain.com[private/dovecot-lmtp], delay=0.04, delays=0.01/0.01/0.02/0, dsn=2.0.0, status=sent (250 2.0.0 <yourname@yourdomain.com> 7gyQI4yjXlxlJAAAUbDMYQ Saved)
Feb  9 09:55:24 mail dovecot: lmtp(9317): Disconnect from local: Successful quit
Feb  9 09:55:24 mail postfix/submission/qmgr[9282]: 89E8946AAB: removed
~~~~
<br>
On this page you got a handle at setting up your own filters for spam. This technique can be applied in many different ways. Folders for shopping, updates, bills, forums, blogs, etc.
