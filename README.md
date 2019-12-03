# Debian Postfix Dovecot MariaDB mailserver tutorial

The aim of this tutorial is to provide ***working*** and ***understandable*** document that will lead you through building your own e-mail server from scratch, which are hard to find with up-to-date informations.

I'm not trying to create a bulletproof server, nor am I trying to create a solution that would reliably serve thousands of enterprise customers, this tutorial is aimed at intermediate Linux user so there will surely be some security weaknesses. You are welcome to update the configuration as you wish.

All packages used in this tutorial are available in Debian repository, there is no need to install any extras, unless you want, of course.

If you have any suggestions, feel free to let me know :-)

## What are we trying to achieve

Our small mailserver will be able to:

- communicate using ***IMAP/SMTP/LMTP*** protocols
- accept only secure connections, ***unencrypted connections will not be supported***
- authenticate users with ***e-mail*** as username and ***password***
- read frequently configuration (domains, users, passwords, quotas, blacklisted phrases) from database
- filter spam
- filter viruses
- limit rates for sending e-mails

## Prerequisities

Our solution will be built upon following packages:

- Debian 10 Buster
- Postfix
- Dovecot
- MariaDB
- certbot
- Rspamd
- ClamAV

This tutorial assumes that you are familiar with linux shell, SQL, editing config files and you can search linux system logs to locate possible problems. I also assumes that you have freshly installed linux server at your disposal and are using account which can sudo command or root account (which is highly insecure, by the way... :-).

It also assumes that you have a pre-configured domain with proper MX records pointing to your server's IP address.

## The steps

To make this tutorial more understandable, i will divide it into following steps, where each of them depends on previous one.

1. Basic configuration using Postfix, Dovecot, MariaDB and certbot.
2. Keywords blocklist
3. Spam filtering using Rspamd
4. Virus filtering using ClamAV

So, let's do it! :-)

### 1. Basic configuration using Postfix, Dovecot, MariaDB and certbot.

