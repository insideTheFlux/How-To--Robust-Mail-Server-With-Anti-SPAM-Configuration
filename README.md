## Introduction
This tutorial will teach you how to set up your own robust email server. We are focusing on a small personal server with up to a few email accounts. After following this guide, you will have a fully functional mail server and you can connect with your favourite client to access, read and send emails. The Anti-Spam configuration will drop unwanted messages.

This tutorial will use **yourdomain.com** as domain name and **mail.yourdomain.com** as hostname for our mail server. The desired email address will be **yourname@yourdomain.com**. We assume that our server has the IP address **1.2.3.4**.

### Software and technologies used
* Postfix v3.3.0 (3.1.0-Ubuntu 16.04.5) as SMTP server
* Dovecot v2.2.33.2 (2.2.22-Ubuntu 16.04.5) as IMAP server
* We will use Unix user accounts and tunnel the SASL authentication through TLS
* Postgrey v1.36 (1.35-Ubuntu 16.04.5) - to reject spam from the beginning
([more about postgrey]( http://postgrey.schweikert.ch/))
* SpamAssassin 3.4.2-Ubuntu 16.04.5(Perl_5.22.1-) - to further combat spam. ([Go there](https://github.com/insideTheFlux/How-To--Robust-Mail-Server-With-Anti-SPAM-Configuration/blob/insideTheFlux-Ubuntu_18-04/SpamAssassin.md))
* Sieve filtering via Dovecot LMTP - Filing done right. ([Go there](https://github.com/insideTheFlux/How-To--Robust-Mail-Server-With-Anti-SPAM-Configuration/blob/insideTheFlux-Ubuntu_18-04/SieveFiltering.md))
* SPF (Sender Policy Framework) validating to reduce spam
([more about SPF](https://www.digitalocean.com/community/articles/how-to-use-an-spf-record-to-prevent-spoofing-improve-e-mail-reliability))
* SPF and [DMARC](http://dmarc.org/) DNS entry to prevent spoofing
* DKIM (Domain Keys Identified Mail) to sign our email messages
([more about DKIM](http://www.dkim.org/))

## Prerequisites

### Personal
Every step will be explained in the tutorial and you will get it running even with minimal Unix knowledge. Nevertheless, you should be used to work on the command-line and know how to use a text editor. Furthermore it takes some Unix skills to administrate the working mail server.
You're invited to follow the links in this tutorial to learn more about the software and techniques used.

While the test bed will be on Ubuntu 18.04, the same should apply to 16.04.5, versions of software will be different. 
After playing with 18.04(which had issues), I've found 16.04_5 to be much more of a stable bed to test on.

* Last time I checked, this fork was 40 commits ahead of [Mike's original master](https://github.com/MikeSkril/How-To--Robust-Mail-Server-With-Anti-SPAM-Configuration). (Feb 28, 2019)
* At this time I recommend starting with 16.04_5
* I recommend Digital Ocean for a VPS.


### System

* A VPS running Ubuntu 18.04/16.04.5(setup will be similar on any Debian based distribution). 
([Get a VPS here](https://www.digitalocean.com/?refcode=79aec8435127))
* Your own FQDN(Fully Qualified Domain Name) domain name.

  
## [Get started here](https://github.com/insideTheFlux/Mail-Server-With-Extras/blob/master/HowTo.md)
