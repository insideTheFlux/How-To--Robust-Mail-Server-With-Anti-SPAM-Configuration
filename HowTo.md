We will be working on a `root` shell and the tutorial will use `vim` as text editor.

## Preparing our system
#### Setting up the host name
~~~~
echo mail.yourdomain.com > /etc/hostname
~~~~

#### Adding our domain to /etc/hosts
~~~~
vim /etc/hosts
~~~~

We add **yourdomain.com** and **mail.yourdomain.com** in the second line
immediately after: `127.0.1.1`
~~~~
127.0.0.1 localhost yourdomain.com mail.yourdomain.com
~~~~

#### Setting up the mailname
~~~~
echo yourdomain.com > /etc/mailname
~~~~

That's the name that will appear on the right side of the '@' in our email address. In this case, **yourdomain.com**.

### Installing required packages

#### Updating the system
~~~~
apt-get update && apt-get upgrade
~~~~

#### Installing
~~~~
apt-get install postfix postfix-policyd-spf-perl postgrey dovecot-core dovecot-imapd opendkim opendkim-tools && postfix stop
~~~~

Hit ‘No’ if asked to create a SSL certificate. Choose `Internet Site` and press 'ok' 2 times when asked by the postfix installer.

## Setting up DNS
We assume that our domain is already setup in the DNS control panel and we see the default DNS records.<p>
1.) Add a hostname under `A record`(Use `@` for hostname), 2.) Will direct to IP, digitalocean starts to create a nice list for you. Pick the right one then hit `Create Record` button.<br><p>
DigitalOcean gives end results to the `Hostname` under the text-area but the `Will Direct to` offers to autofill with a list of servers that are available. Pick the right one.
![DNS default](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/DNS_DO_edit.png?raw=true)


### Setting up the A record - `Will Direct To` autofills.
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/A_record_edit.png?raw=true)

(There is a `'dot'` after the domain name)


### Setting up the MX record
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/MX_record_edit.png?raw=true)

(There is a `'dot'` after the domain name)
**Digital ocean repeats what you're typing below the text-area**


### Setting up the SPF record

We create a new TXT record
~~~~
"v=spf1 a mx ip4:1.2.3.4 -all"
~~~~

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/SPF_record_edit.png?raw=true)


The SPF record protects from email spoofing. It will simply tell other mail servers that only our server is authorized to send emails for **yourdomain.com** ([more aboutSPF](https://www.digitalocean.com/community/articles/how-to-use-an-spf-record-to-prevent-spoofing-improve-e-mail-reliability)).

### Setting up the DMARC record

We create a new TXT record named **`_dmarc.yourdomain.com.`**
(There is a ‘dot’ after the domain name)
~~~~
"v=DMARC1; p=quarantine; rua=mailto:postmaster@yourdomain.com"
~~~~
**`rua`** Reporting URI of aggregate reports. 

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/DMARC_record_edit.png?raw=true)
(There is a `'dot'` after the domain name)

#### By this point there should be an entry in the PTR list
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/PTR_record_edit.png?raw=true)

### Our configuration should look similar to this
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/DNSRecords_FullSetup_edit.png?raw=true)


It will take a while to propagate the new configuration throughout the entire internet. While you wait, take a look at [How to Manage DNS Records](https://www.digitalocean.com/docs/networking/dns/how-to/manage-records/)


## Generating SSL certificates

There are different ways to generate a SSL certificate. The tutorial will use a self signed certificate but we could also use a [CAcert](http://en.wikipedia.org/wiki/Certificate_authority), which can be obtained for free from [startssl](https://www.startssl.com) & [letsencrypt](https://letsencrypt.org/).
Either ways, the connection from our email client to the mail server will be encrypted.

### Creating the certificate
~~~~
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/mail.yourdomain.key -out /etc/ssl/certs/mail.yourdomain.pem
~~~~

We will be asked to answer a few questions. It is important that we enter **mail.yourdomain.com** as `Common name`.
~~~~
Generating a 2048 bit RSA private key
..........+++
..................................................................+++
writing new private key to '/etc/ssl/private/mail.yourdomain.key'
- -----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
- -----
Country Name (2 letter code) [AU]:AT
State or Province Name (full name) [Some-State]:Styria
Locality Name (eg, city) []:Vienna
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:mail.yourdomain.com
Email Address []:postmaster@yourdomain.com
~~~~

## Postfix Configuration
### Backing up the configuration files

~~~~
cp /etc/postfix/master.cf /etc/postfix/master.cf_orig &&
cp /etc/postfix/main.cf /etc/postfix/main.cf_orig
~~~~

### Configuration in main.cf
~~~~
vim /etc/postfix/main.cf
~~~~

We empty the file and add our configuration

~~~~
# Begin config
smtpd_banner= $myhostname ESMTP Your server message here
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h
delay_warning_time=3h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 2 on
# fresh installs.
compatibility_level = 2

# SSL/TLS
smtpd_tls_auth_only = yes
smtpd_tls_cert_file=/etc/letsencrypt/live/mail.mydomain.com/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/mail.mydomain.com/privkey.pem
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_tls_security_level=encrypt
smtp_tls_security_level=encrypt

myhostname = mail.mydomain.com
mydestination = $myhostname, mydomain.com, mail.mydomain.com, localhost.mydomain.com, localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
maximal_queue_lifetime=2d
bounce_queue_lifetime=1d
syslog_name=postfix/submission

# Aliases / Recipients
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
local_recipient_maps = proxy:unix:passwd.byname $alias_maps

smtpd_sasl_type=dovecot
smtpd_sasl_path=private/auth
smtpd_sasl_auth_enable=no


policy-spf_time_limit = 3600s
smtpd_helo_required = yes

header_checks = regexp:/etc/postfix/outgoing_mail_header_filters
mime_header_checks = regexp:/etc/postfix/outgoing_mail_header_filters

smtpd_recipient_restrictions =
 permit_mynetworks
 permit_sasl_authenticated
 reject_rbl_client zen.spamhaus.org
 reject_unlisted_recipient
 reject_unauth_destination
 check_policy_service unix:private/policy-spf
 check_policy_service inet:127.0.0.1:10023

smtpd_helo_restrictions =
 permit_mynetworks
 reject_non_fqdn_helo_hostname
 reject_invalid_helo_hostname

smtpd_client_restrictions=
 permit_mynetworks
 permit_sasl_authenticated
 reject_unknown_client_hostname

smtpd_relay_restrictions =
 permit_mynetworks
 permit_sasl_authenticated
 reject_unauth_destination

smtpd_data_restrictions =
 reject_unauth_pipelining

#DKIM
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
~~~~

### Configuration in master.cf

~~~~
vim /etc/postfix/master.cf
~~~~

This line must be active:
**`Be mindful of the spacing on the second line`**
~~~~ 
smtp inet n - y - - smtpd
  -o smtpd_sasl_auth_enable=yes
~~~~

We uncomment the following:
**`Be mindful of the spacing on the second line`**
~~~~
submission inet n - - - - smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_milters=inet:127.0.0.1:8891
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
~~~~

and add these 2 lines below, at the end of the file 
(there are 2 whitespaces before the second line).

~~~~
policy-spf unix - n n - - spawn
  user=nobody argv=/usr/sbin/postfix-policyd-spf-perl
~~~~

Save the file, exit.

---

Create a file **`outgoing_mail_header_filters`** in **`/etc/postfix`**
~~~~
vim outgoing_mail_header_filters
~~~~

Add the following inside the file:
~~~~
# Remove the first line of the Received: header. Note that we cannot fully remove the Received: header
# because OpenDKIM requires that a header be present when signing outbound mail. The first line is
# where the user's home IP address would be.
/^\s*Received:[^\n]*(.*)/     REPLACE Received: from authenticated-user (mail.yourdomain.com [YOUR IP ADDRESS])

# Remove other typically private information.
/^\s*User-Agent:/        IGNORE
/^\s*X-Enigmail:/        IGNORE
/^\s*X-Mailer:/          IGNORE
/^\s*X-Originating-IP:/  IGNORE
/^\s*X-Pgp-Agent:/       IGNORE

# The Mime-Version header can leak the user agent too, e.g. in Mime-Version: 1.0 (Mac OS X Mail 8.1 \(2010.6\)).
/^\s*(Mime-Version:\s*[0-9\.]+)\s.+/  REPLACE $1
~~~~
Save the file, exit.

#### This is how our headers looked at first.
---
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/beforeheaders.png?raw=true)

#### Now showing less info, a disguise.
---
![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/afterheaders.png?raw=true)

---
More information about the postfix configuration parameters can be found [here](http://www.postfix.org/postconf.5.html).

## Dovecot Configuration

### Backing up the configuration file

~~~~
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf_orig
~~~~

### Configuration in dovecot.conf

~~~~
vim /etc/dovecot/dovecot.conf
~~~~

We empty the file and add our configuration.

~~~~
disable_plaintext_auth = no
mail_privileged_group = mail
mail_location = mbox:~/mail:INBOX=/var/mail/%u
userdb {
driver = passwd
}

passdb {
args = %s
driver = pam
}
protocols = imap

# Check out :: http://wiki2.dovecot.org/MailboxSettings
namespace inbox {

                inbox = yes

        mailbox Spam {
                special_use = \Junk
                auto = subscribe
                }
        mailbox Drafts {
                special_use = \Drafts
                auto = subscribe
                }
        mailbox Sent {
                special_use = \Sent
                auto = subscribe # autocreate and autosubscribe the Sent mailbox
                }
        mailbox Trash {
                special_use = \Trash
                auto = subscribe
                }


        mailbox "Sent Messages" {
                special_use = \Sent
        }
        mailbox Junk {
                special_use = \Junk
        }

}

service auth {
unix_listener /var/spool/postfix/private/auth {
group = postfix
mode = 0660
user = postfix
}
}
ssl=required
ssl_cert = </etc/ssl/certs/mail.yourdomain.pem
ssl_key = </etc/ssl/private/mail.yourdomain.key
ssl_protocols = !SSLv2 !SSLv3 !TLSv1
ssl_cipher_list = ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ALL:!LOW:!SSLv2:!EXP:!aNULL
ssl_prefer_server_ciphers = yes
ssl_dh_parameters_length = 2048
~~~~
More information about the dovecot configuration parameters can be found [here](http://wiki2.dovecot.org/).

### Restarting dovecot

~~~~
service dovecot restart
~~~~

## Adding our mail user

Now we add the user for our mail account. In this example our email
address will be **yourname@yourdomain.com**.
We can add as many users/email addresses as we want.

~~~~
adduser --gecos '' --shell /bin/false yourname
~~~~

## Setting Aliases

~~~~
vim /etc/aliases
~~~~

The configuration can be adapted to your needs.
Here we tell postfix to forward all messages addressed to **aliases** to the mailbox of user: **yourname**.

~~~~
mailer-daemon: postmaster
postmaster: root
nobody: root
hostmaster: root
usenet: root
news: root
webmaster: root
www: root
ftp: root
abuse: root
noc: root
security: root
admin: root
root: yourname
~~~~

### Compiling the alias file

~~~~
newaliases
~~~~

#### Don't forget to set this to **`yes`**
After adding your users, set this to yes in dovecot.conf
~~~~
disable_plaintext_auth = yes
~~~~
~~~~
service dovecot restart
~~~~

## Setting up DKIM (Domain Keys Identified Mail)

### Creating the directory and files

~~~~
mkdir /etc/opendkim
~~~~

~~~~
vim /etc/opendkim/KeyTable
~~~~

We enter the following line (all in the first line):
~~~~
default._domainkey.mail.yourdomain.com mail.yourdomain.com:default:/etc/opendkim/default.private
~~~~

~~~~
vim /etc/opendkim/SigningTable
~~~~

We enter the following line:
~~~~
*@yourdomain.com default._domainkey.mail.yourdomain.com
~~~~

### Generating the key pair
~~~~
opendkim-genkey -b 2048 -s default -d mail.yourdomain.com -D /etc/opendkim
~~~~

### Changing ownership of the private key file

~~~~
chown opendkim:opendkim /etc/opendkim/default.private
~~~~

### Configuration in opendkim

~~~~
vim /etc/default/opendkim
~~~~

### We add the line below to the configuration
~~~~
SOCKET="inet:8891@localhost"
~~~~

### Configuration in opendkim.conf

~~~~
vim /etc/opendkim.conf
~~~~

### We fill the file with our configuration
~~~~
Syslog yes
SyslogSuccess Yes
LogWhy yes

UMask 002

KeyTable refile:/etc/opendkim/KeyTable
SigningTable refile:/etc/opendkim/SigningTable
Selector default
X-Header no
SignatureAlgorithm rsa-sha256

Canonicalization relaxed/simple
Mode sv
AutoRestart Yes
AutoRestartRate 5/1h
InternalHosts 127.0.0.1, localhost, yourdomain.com, mail.yourdomain.com

OversignHeaders From
~~~~

### Setting the DKIM DNS record

~~~~
cat /etc/opendkim/default.txt
~~~~

This is our public key which will be used to verify the signature in our emails.

~~~~
default._domainkey IN TXT "v=DKIM1; k=rsa;
p=MIGfHL0GCSqGSIb3DQESYJFOA4GNADCBiQKBgQDS+vPyWRs7w32xomf2oZIexmS2TuQAXKPiQ3AXn4j25NOReXdgKxIqAwl3O7dQtgluWw+TH85Mrbmx5UgwaaLenj9cfe2IRvx7hvkj7+6i0XQqrWqZlMw+QAJxAGhfa/GVTYa+/7PFWfXLoqoBW5arE+wO20O2uw5Ik62HjkKZbQIDAQAB" ; ----- DKIM key default for mail.yourdomain.com
~~~~

#### Now we add a new TXT record:

As 'name' we put (there is a 'dot' at the end):

~~~~
default._domainkey.mail.yourdomain.com.
~~~~

As 'text' we add (use your public key from /etc/opendkim/default.txt):
~~~~
"v=DKIM1; k=rsa; p=MIGfHL0GCSqGSIb3DQESYJFOA4GNADCBiQKBgQDS+vPyWRs7w32xomf2oZIexmS2TuQAXKPiQ3AXn4j25NOReXdgKxIqAwl3O7dQtgluWw+TH85Mrbmx5UgwaaLenj9cfe2IRvx7hvkj7+6i0XQqrWqZlMw+QAJxAGhfa/GVTYa+/7PFWfXLoqoBW5arE+wO20O2uw5Ik62HjkKZbQIDAQAB"
~~~~

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/DKIM_DNS_record_edit.png?raw=true)


It will take a while to propagate the new configuration throughout the entire internet.

## Starting postfix and restarting opendkim

~~~~
postfix start && service opendkim restart
~~~~

## Issues that might arise while trying to run `opendkim` on Ubuntu 18.04

Issues with opendkim starting on 18.04/ Opendkim v2.11.0: <br>
Running tail -f /var/log/syslog -- this is the error you might see:<br>
These issues will get fixed later in Ubuntu 18.04 but for now we frankenstein-it.
~~~~
Jan 26 00:43:59 mail systemd[1]: opendkim.service: Control process exited, code=exited status=78
Jan 26 00:43:59 mail systemd[1]: opendkim.service: Failed with result 'exit-code'.
Jan 26 00:43:59 mail systemd[1]: Failed to start OpenDKIM DomainKeys Identified Mail (DKIM) Milter.
~~~~
Inside the following file: 
~~~~
vim /lib/opendkim/opendkim.service.generate
~~~~
You'll find this block:
~~~~
if [ -s $SERVICE ] ; then
        mkdir -p /etc/systemd/system/$NAME.service.d
        cp $SERVICE /etc/systemd/system/$NAME.service.d/override.conf
fi
~~~~
Change it so it looks like this, save the file, exit.
~~~~
if [ -s $SERVICE ] ; then
        mkdir -p /etc/systemd/system/$NAME.service.d
        cp $SERVICE /etc/systemd/system/$NAME.service.d/override.conf
        install -m 644 $SERVICE /etc/systemd/system/$NAME.service.d/override.conf
fi
~~~~
## Run: 
~~~~
/lib/opendkim/./opendkim.service.generate
~~~~
~~~~
systemctl daemon-reload
~~~~
~~~~
service opendkim restart
~~~~

This should take care of the problem with opendkim starting up. As I said, there's a bug fix in the works as of January 2019.


## Connecting with a Client

Now we will connect with our email client. In the tutorial we use Thunderbird as client.
Just add a new email account in Thunderbird and it will auto-detect the servers configuration.

The configuration should look like this:

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/server_autosetup.png?raw=true)

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/server_manual_config_edit.png?raw=true)


## Testing the mail server

It's time to test our server. Let's send and receive emails within Thunderbird.
We can observe the ongoing transactions and spot possible errors by monitoring our syslog.
~~~~
tail -f /var/log/syslog
~~~~

## You might notice another error in syslog, again:

This time it happens during sending emails, nothing gets sent. Here's an image:

![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/sending_errors_edit.png?raw=true)


![](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/edited/errors_thunderbird_edit.png?raw=true)


It appears to be a permisions issue: default.private is owned by opendkim:opendkim user/group -- I think this can also be set to root:root but let's just apply:
~~~~
chown root:opendkim /etc/opendkim/default.private
~~~~


We can test our SPF and DKIM configuration [here](https://mxtoolbox.com/spf.aspx) and DMARC [here](https://mxtoolbox.com/dmarc.aspx) or DKIM mxtoolbox [here](https://mxtoolbox.com/dkim.aspx)
~~~~
domain name: mail.yourdomain.com
selector: default
~~~~

Some more testing can be done [here](https://www.wormly.com/test_smtp_server), or [here](http://www.emailsecuritygrader.com/), or [here](https://www.mail-tester.com/).

  
### Getting a certificate from Let's Encrypt, moving away from self signed ones.
There's nothing wrong with using a self signed certificate, other than having to have Thunderbird add exceptions for them at the beginning. Thunderbird's ability to read your server stats is exceptional, I think other mail clients would break trying to get your server details in order.<br>
With that in mind, having a more accepted certificate would reduce the need for Exceptions and maybe allow other mail clients to function properly. The catch? You'll need to renew every ninety days, it can be done automatically too(I'll leave the magic for that up to you to find), the self signed lasts up-to one year per this page.<br>

Installing Let's Encrypt Certbot:
~~~~
  add-apt-repository ppa:certbot/certbot
~~~~
~~~~
  apt-get update
~~~~
~~~~
  apt-get install certbot
~~~~
This option won't need a webserver running, certbot will handle it, port 80 should be open incase you're using a firewall.
When asked, enter a valid email address that **`certbot`**  can remind you to renew before ninety day period is over.
~~~~
  certbot certonly --standalone --preferred-challenges http -d mail.yourdomain.com
~~~~

~~~~
  - Congratulations! Your certificate and chain have been saved at:   /etc/letsencrypt/live/mail.yourdomain.com/fullchain.pem
   Your key file has been saved at:   /etc/letsencrypt/live/mail.yourdomain.com/privkey.pem
   Your cert will expire on 2019-05-01. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
~~~~
Don't forget to apply those settings to your **` main.cf `** here:
  ~~~~
  smtpd_tls_cert_file=/etc/letsencrypt/live/mail.yourdomain.com/fullchain.pem
  smtpd_tls_key_file=/etc/letsencrypt/live/mail.yourdomain.com/privkey.pem
  ~~~~
And your **` dovecot.conf `** file as well
  ~~~~
  ssl_cert = </etc/letsencrypt/live/mail.yourdomain.com/fullchain.pem
  ssl_key = </etc/letsencrypt/live/mail.yourdomain.com/privkey.pem
  ~~~~
Restart both services
~~~~
service postfix restart
service dovecot restart
~~~~


Don't forget that your certificate is only valid for ninety days, you will need to renew it. Manually renew before expiration date.
Dry run:
~~~~
certbot renew --dry-run
~~~~

True renew:
~~~~
certbot renew
~~~~

Restart both services
~~~~
service postfix restart
service dovecot restart
~~~~


### Postgrey in action

Pass:
~~~~
Feb  1 23:40:49 mail postgrey[1583]: action=pass, reason=triplet found, delay=492,
client_name=otherserver.com, client_address=xxx.xxx.xx.xxx, 
sender=user@otherserver.com, recipient=youruser@yourdomain.com
~~~~

Reject at first try:

~~~~
Feb  1 23:32:37 mail postgrey[1583]: action=greylist, reason=new, client_name=otherserver.com, 
client_address=xxx.xxx.xx.x, sender=otheruser@otherserver.com, recipient=youruser@yourdomain.com
Feb  1 23:32:37 mail postfix/submission/smtpd[1907]: NOQUEUE: reject: 
RCPT from otherserver[xxx.xxx.xx.x]: 450 4.2.0 <youruser@yourdomain.com>: Recipient address rejected: 
Greylisted, see http://postgrey.schweikert.ch/help/yourdomain.com.html; 
from=<otheruser@otherserver.com> to=<youruser@yourdomain.com> proto=ESMTP helo=<otherserver.com>
~~~~

### DKIM in action

Sending an email:
~~~~
Feb  2 05:51:09 mail opendkim[1477]: C19C63F5CA: DKIM-Signature field added (s=default, d=mail.yourdomain.com)
~~~~

### SPF in action

Pass:
~~~~
Jan 31 21:44:17 mail postfix/policy-spf[5248]: Policy action=PREPEND 
Received-SPF: pass (google.com: Sender is authorized to use 'noreply-dmarc-support@google.com'
in 'mfrom' identity (mechanism 'include:_spf.google.com' matched)) receiver=yourdomain.com; 
identity=mailfrom; envelope-from="noreply-dmarc-support@google.com"; 
helo=mail-it1-f202.google.com; client-ip=209.85.166.202
~~~~

Reject:
~~~~
May 12 16:15:25 mail postfix/policy-spf[20469]: Policy action=550 Please see http://www.openspf.net/Why?s=mfrom;id=someone 4yourdomain.com;ip=184.72.226.23;r=yourdomain.com
May 12 16:15:25 mail postfix/submission/smtpd[20463]: NOQUEUE: reject:
RCPT from node-mec2.wormly.com[184.72.226.23]: 550 5.7.1 <yourname@
yourdomain.com >: Recipient address rejected: Please see http://www.openspf.net/Why?s=mfrom;id=someone%
yourdomain.com;ip=184.72.226.23;r=yourdomain.com;
from=<someone@yourdomain.com> to=<yourname@yourdomain.com> proto=ESMTP
helo=
~~~~

## Congratulations

You did it! You setup our own mail server and do not depend anymore on
any email provider.

Our anti-spam setup should filter out 99% of all unwanted messages and
won't drop a legitimate email. On the other hand, messages from our
server should not be declared as spam as it has a valid SPF record and
we are signing our messages.

### Get SpamAssassin installed - reduce spam

## [SpamAssassin filter for Postfix](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/SpamAssassin.md)

## Useful for troubleshooting

### Verify DNS

Check reverse entry
~~~~
dig -x 1.2.3.4 +short
~~~~

Check MX record
~~~~
dig yourdomain.com MX +short
~~~~

### Verify if ports are listening

~~~~
netstat -tulpen|grep LISTEN
~~~~

### Enable verbose logging for postfix in /etc/postfix/master.cf
Just add the `v` at the end of the smtp line.

~~~~
smtp inet n - - - - smtpd -v
~~~~

### Check the syslog with
~~~~
tail -f /var/log/syslog
~~~~
and observe the logging.
