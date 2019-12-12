# Debian Postfix Dovecot MariaDB certbot mailserver tutorial

The goal of this tutorial is to provide ***working*** and ***understandable*** document that will guide you through building your own e-mail server from scratch, which are hard to find with up-to-date informations.

I'm not trying to create a bulletproof server, nor am I trying to create a solution that would reliably serve thousands of enterprise customers, this tutorial is aimed at intermediate Linux user so there will surely be some security weaknesses. You are welcome to update the configuration as you wish.

All packages used in this tutorial are available in Debian repository, there is no need to install any extras, unless you want, of course.

If you have any suggestions, feel free to let me know, or better - fork master branch, make changes and create a pull request :-)

## What are we trying to achieve

Our small mailserver will be able to:

- communicate using ***IMAP/SMTP/LMTP*** protocols
- accept only secure connections, ***unencrypted connections will not be supported***
- authenticate users with ***e-mail*** as username and ***password***
- read frequently changed configuration (domains, users, passwords, quotas, blacklisted phrases) from database, so we can make quick changes
- limit rates for sending e-mails, ask some RBLs if e-mail should be blocked
- filter spam
- filter viruses

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

It also assumes that you have a pre-configured domain with proper MX records pointing to your server's IP address. In this tutorial, i will use following settings:

    Domain: mydomain.com
    Server's FQDN: mail.mydomain.com
    MariaDB user: mailuser
    MariaDB password: mailpassword
    Database name: mailserver
    E-mail: mymailbox@mydomain.com
    E-mail password: mailboxpassword

## The steps

To make this tutorial more understandable, i will divide it into following steps, where each of them depends on previous one.

1. [Basic configuration using Postfix, Dovecot, MariaDB and certbot](#1-basic-configuration-using-postfix-dovecot-mariadb-and-certbot)
2. [RBL checks](#2-rbl-checks)
3. [Keywords blacklist](#3-keywords-blacklist)
4. Spam filtering using Rspamd
5. Virus filtering using ClamAV

So, let's do it! :-)

### 1. Basic configuration using Postfix, Dovecot, MariaDB and certbot

Install required packages

    sudo apt-get install mariadb-server postfix postfix-mysql dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd dovecot-mysql certbot

You will not be prompted to enter a password for the root MariaDB user for recent versions of MariaDB. This is because on Debian and Ubuntu, MariaDB now uses either the `unix_socket` or `auth_socket` authorization plugin by default. This authorization scheme allows you to log in to the database’s root user as long as you are connecting from the Linux root user on localhost.

When prompted, select Internet Site as the type of mail server the Postfix installer should configure. The System Mail Name should be the FQDN, in this case `mail.mydomain.com`.

#### Install certbot and obtain a certificate

Certbot installation is pretty easy, so obtaining a certificate should not be a big hurdle. First obtain a certificate for your domain. If you are nto running any webserver on your installation, choose `Spin up a temporary webserver` when prompted how would you like to authenticate with the ACME CA:

    sudo certbot certonly -d mail.mydomain.com

Next, try renewing the certificate. It is just the dry-run so nothing is actually renewed, but certbot will add a command to autorenew your certificates, which will handle the renewals itself later.

    sudo certbot renew --dry-run

#### Set up and populate the database

Create database, user and se privileges:

    CREATE DATABASE mailserver;

    GRANT SELECT, EXECUTE ON mailserver.* TO 'mailuser'@'127.0.0.1' IDENTIFIED BY 'mailuserpass';
    FLUSH PRIVILEGES;

Create table for domains that will be server by our server:

    CREATE TABLE `maildomains` (
    `id` mediumint(5) NOT NULL auto_increment,
    `name` varchar(64) NOT NULL,
    `enabled` tinyint(1) unsigned NOT NULL DEFAULT '1',
    PRIMARY KEY (`id`),
    INDEX `name` (`name`),
    INDEX `enabled` (`enabled`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Create table that will contain list of all users / mailboxes:

    CREATE TABLE `mailusers` (
    `id` int(8) NOT NULL auto_increment,
    `domain_id` mediumint(5) NOT NULL,
    `password` varchar(106) NOT NULL,
    `email` varchar(255) NOT NULL,
    `quota` bigint(20) unsigned NOT NULL DEFAULT '0',
    `enabled` tinyint(1) unsigned NOT NULL DEFAULT '1',
    PRIMARY KEY (`id`),
    UNIQUE KEY `email` (`email`),
    INDEX `enabled` (`enabled`),
    FOREIGN KEY (domain_id) REFERENCES maildomains(id) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Create table for e-mail aliases:

    CREATE TABLE `mailaliases` (
    `id` int(8) NOT NULL auto_increment,
    `domain_id` mediumint(5) NOT NULL,
    `source` varchar(255) NOT NULL,
    `destination` varchar(255) NOT NULL,
    `enabled` tinyint(1) unsigned NOT NULL DEFAULT '1',
    PRIMARY KEY (`id`),
    INDEX `source` (`source`),
    INDEX `enabled` (`enabled`),
    FOREIGN KEY (domain_id) REFERENCES maildomains(id) ON DELETE CASCADE ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Now let's add some data:

    INSERT INTO `maildomains`
    (`id` ,`name`)
    VALUES
    ('1', 'mydomain.com');

    INSERT INTO `mailusers`
    (`id`, `domain_id`, `password` , `email`)
    VALUES
    ('1', '1', ENCRYPT('mailboxpassword', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'mymailbox@mydomain.com');

    INSERT INTO `mailaliases`
    (`id`, `domain_id`, `source`, `destination`)
    VALUES
    ('1', '1', '@mydomain.com', 'mymailbox@mydomain.com');

#### Configure Postfix

Postfix is a Mail Transfer Agent (MTA) that relays mail between the Linode and the internet. It is highly configurable, allowing for great flexibility. TIn this guide I maintain many of Posfix’s default configuration values, thus it's highly recommended to dig deeper into documentation and adjust the setting more specifically.

Primary configuration file used by Postfix is `main.cf`. First we will make a backup of this file in case that we need to revert the whole configuration.

    sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.orig

Now lets edit the `/etc/postfix/main.cf` so it will look like this:

    # See /usr/share/postfix/main.cf.dist for a commented, more complete version

    # Debian specific:  Specifying a file name will cause the first
    # line of that file to be used as the name.  The Debian default
    # is /etc/mailname.
    #myorigin = /etc/mailname

    smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)
    biff = no

    # appending .domain is the MUA's job.
    append_dot_mydomain = no

    # Uncomment the next line to generate "delayed mail" warnings
    #delay_warning_time = 4h

    readme_directory = no

    # TLS parameters
    smtpd_tls_cert_file=/etc/letsencrypt/live/mail.mydomain.com/fullchain.pem
    smtpd_tls_key_file=/etc/letsencrypt/live/mail.mydomain.com/privkey.pem
    smtpd_use_tls=yes
    smtpd_tls_auth_only = yes
    smtp_tls_security_level = may
    smtpd_tls_security_level = may
    smtpd_sasl_security_options = noanonymous, noplaintext
    smtpd_sasl_tls_security_options = noanonymous

    # Authentication
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
    smtpd_sasl_auth_enable = yes

    # See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
    # information on enabling SSL in the smtp client.

    # Restrictions
    smtpd_helo_restrictions =
            permit_mynetworks,
            permit_sasl_authenticated,
            reject_invalid_helo_hostname,
            reject_non_fqdn_helo_hostname
    smtpd_recipient_restrictions =
            permit_mynetworks,
            permit_sasl_authenticated,
            reject_non_fqdn_recipient,
            reject_unknown_recipient_domain,
            reject_unlisted_recipient,
            reject_unauth_destination,
            reject_unauth_pipelining
    smtpd_sender_restrictions =
            permit_mynetworks,
            permit_sasl_authenticated,
            reject_non_fqdn_sender,
            reject_unknown_sender_domain
    smtpd_relay_restrictions =
            permit_mynetworks,
            permit_sasl_authenticated,
            defer_unauth_destination

    # See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
    # information on enabling SSL in the smtp client.

    myhostname = mail.mydomain.com
    alias_maps = hash:/etc/aliases
    alias_database = hash:/etc/aliases
    mydomain = mydomain.com
    myorigin = $mydomain
    mydestination = localhost
    relayhost =
    mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
    mailbox_size_limit = 0
    recipient_delimiter = +
    inet_interfaces = all
    inet_protocols = all

    # Handing off local delivery to Dovecot's LMTP, and telling it where to store mail
    virtual_transport = lmtp:unix:private/dovecot-lmtp

    # Virtual domains, users, and aliases
    virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
    virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
    virtual_alias_maps = mysql:/etc/postfix/mysql-virtual-alias-maps.cf,
            mysql:/etc/postfix/mysql-virtual-email2email.cf

    # Even more Restrictions and MTA params
    disable_vrfy_command = yes
    strict_rfc821_envelopes = yes
    #smtpd_etrn_restrictions = reject
    #smtpd_reject_unlisted_sender = yes
    #smtpd_reject_unlisted_recipient = yes
    smtpd_delay_reject = yes
    smtpd_helo_required = yes
    smtp_always_send_ehlo = yes
    #smtpd_hard_error_limit = 1
    smtpd_timeout = 30s
    smtp_helo_timeout = 15s
    smtp_rcpt_timeout = 15s
    smtpd_recipient_limit = 40
    minimal_backoff_time = 180s
    maximal_backoff_time = 3h

    # Reply Rejection Codes
    invalid_hostname_reject_code = 550
    non_fqdn_reject_code = 550
    unknown_address_reject_code = 550
    unknown_client_reject_code = 550
    unknown_hostname_reject_code = 550
    unverified_recipient_reject_code = 550
    unverified_sender_reject_code = 550

The `main.cf` file declares the location of `virtual_mailbox_domains`, `virtual_mailbox_maps`, and `virtual_alias_maps` files. These files contain the connection information for the MySQL lookup tables created in the MySQL section of this tutorial. Postfix will use this data to identify all domains, corresponding mailboxes, and valid users.

Create the file `/etc/postfix/mysql-virtual-mailbox-domains.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT 1 FROM maildomains WHERE name='%s' AND enabled=1

Create the file `/etc/postfix/mysql-virtual-mailbox-maps.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT 1 FROM mailusers WHERE email='%s' AND enabled=1

Create the file `/etc/postfix/mysql-virtual-alias-maps.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT destination FROM mailaliases WHERE source='%s' AND enabled=1

Create the file `/etc/postfix/mysql-virtual-email2email.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT email FROM mailusers WHERE email='%s' AND enabled=1

#### Set up Postfix master

Postfix’s master program starts and monitors all of Postfix’s processes. The configuration file `master.cf` lists all programs and information on how they should be started. Lets begin with creating a backup.

    sudo cp /etc/postfix/master.cf /etc/postfix/master.cf.orig

Now edit `/etc/postfix/master.cf` to contain the values in the excerpt example. The rest of the file can remain unchanged:

    #
    # Postfix master process configuration file.  For details on the format
    # of the file, see the master(5) manual page (command: "man 5 master" or
    # on-line: http://www.postfix.org/master.5.html).
    #
    # Do not forget to execute "postfix reload" after editing this file.
    #
    # ==========================================================================
    # service type  private unpriv  chroot  wakeup  maxproc command + args
    #               (yes)   (yes)   (yes)    (never) (100)
    # ==========================================================================
    smtp      inet  n       -       n       -       -       smtpd
    #smtp      inet  n       -       -       -       1       postscreen
    #smtpd     pass  -       -       -       -       -       smtpd
    #dnsblog   unix  -       -       -       -       0       dnsblog
    #tlsproxy  unix  -       -       -       -       0       tlsproxy
    submission inet n       -       y      -       -       smtpd
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_sasl_type=dovecot
    -o smtpd_sasl_path=private/auth
    -o smtpd_reject_unlisted_recipient=no
    -o smtpd_client_restrictions=permit_sasl_authenticated,reject
    -o milter_macro_daemon_name=ORIGINATING
    smtps     inet  n       -       -       -       -       smtpd
    -o syslog_name=postfix/smtps
    -o smtpd_tls_wrappermode=yes
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_sasl_type=dovecot
    -o smtpd_sasl_path=private/auth
    -o smtpd_client_restrictions=permit_sasl_authenticated,reject
    -o milter_macro_daemon_name=ORIGINATING
    ...

Restrict the permissions of the `/etc/postfix` directory to allow only its owner and the corresponding group:

    sudo chmod -R o-rwx /etc/postfix

#### Configure Dovecot

Dovecot is the _Mail Delivery Agent_ (MDA) which is passed messages from Postfix and delivers them to a virtual mailbox. In this section, configure Dovecot to force users to use SSL when they connect so that their passwords are never sent to the server in plain text.

First back up all of the configuration files so you can easily revert back to them if needed:

    sudo cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.orig
    sudo cp /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.orig
    sudo cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.orig
    sudo cp /etc/dovecot/dovecot-sql.conf.ext /etc/dovecot/dovecot-sql.conf.ext.orig
    sudo cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.orig
    sudo cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.orig

Edit the `/etc/dovecot/dovecot.conf` file. Add `protocols = imap pop3 lmtp` to the `# Enable installed protocols` section of the file:

    ## Dovecot configuration file
    ...
    # Enable installed protocols
    !include_try /usr/share/dovecot/protocols.d/*.protocol
    protocols = imap pop3 lmtp
    ...
    postmaster_address=postmaster@mydomain.com

Edit the `/etc/dovecot/conf.d/10-mail.conf` file. This file controls how Dovecot interacts with the server’s file system to store and retrieve messages:

Modify the following variables within the configuration file:

    ...
    mail_location = maildir:/var/mail/vhosts/%d/%n/
    ...
    mail_privileged_group = vmail
    ...

Create the `/var/mail/vhosts/` directory. This directory will serve as storage for mail sent to your domain.

    sudo mkdir -p /var/mail/vhosts

Create the `vmail` group with ID `5000`. Add a new user `vmail` to the `vmail` group. This system user will read mail from the server. Also change the owner of the /var/mail/ folder and its contents to belong to vmail:

    sudo groupadd -g 5000 vmail
    sudo useradd -g vmail -u 5000 vmail -d /var/mail
    sudo chown -R vmail:vmail /var/mail

In case of our settings, the full path where our mail is stored is ***/var/mail/vhosts/mydomain.com/mymailbox***.

All required subdirectories will be created automatically by Dovecot, which is a huge benefit. Still, there is one drawback - when you delete mailbox from database, the mailbox folder will not be deleted accordingly - you have to do it manually.

Edit the user authentication file, located in `/etc/dovecot/conf.d/10-auth.conf`. Uncomment the following variables and replace with the file excerpt’s example values:

    ...
    disable_plaintext_auth = yes
    ...
    auth_mechanisms = plain login
    ...
    !include auth-system.conf.ext
    ...
    !include auth-sql.conf.ext
    ...

Edit the `/etc/dovecot/conf.d/auth-sql.conf.ext` file with authentication and storage information. Ensure your file contains the following lines. Make sure the `passdb` section is uncommented, that the `userdb` section that uses the sql driver is uncommented and update with the right `args`, and comment out the userdb section that uses the `static` driver:

    ...
    passdb {
        driver = sql
        args = /etc/dovecot/dovecot-sql.conf.ext
    }
    ...
    userdb {
        driver = sql
        args = /etc/dovecot/dovecot-sql.conf.ext
    }
    ...
    #userdb {
    #    driver = static
    #    args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
    #}
    ...

Update the `/etc/dovecot/dovecot-sql.conf.ext` file with your MySQL connection information. Uncomment the following variables and replace the values with the excerpt example.

    ...
    driver = mysql
    ...
    connect = host=127.0.0.1 dbname=mailserver user=mailuser password=mailuserpass
    ...
    default_pass_scheme = SHA512-CRYPT
    ...
    password_query = SELECT u.email AS user, u.password, CONCAT('*:bytes=', u.quota) AS userdb_quota_rule FROM mailusers u INNER JOIN maildomains d ON d.id=u.domain_id WHERE u.email='%u' AND u.enabled=1 AND d.enabled=1;
    ...
    user_query = SELECT CONCAT('/var/mail/vhosts/', SUBSTRING_INDEX(u.email, '@', -1), '/', SUBSTRING_INDEX(u.email, '@', 1)) as home, 'vmail' as uid, 'vmail' as gid, CONCAT('*:bytes=', u.quota) AS quota_rule FROM mailusers u INNER JOIN maildomains d ON d.id=u.domain_id WHERE u.email='%u' AND u.enabled=1 AND d.enabled=1;
    ...

The `password_query` variable uses data in the `mailusers` table as the username credential for an email account. The `user_query` variable uses data in the `mailusers` table for primary verification whether user exists in the first place.

You can easily enabled / disable users or whole domains by simply turning value in `enabled` column from 1 to 0. Both queries also uses `quota` as user quota in bytes, so you can easily edit quotas by simply changing this entry.

Change the owner and group of the `/etc/dovecot/` directory to `vmail` and `dovecot`:

    sudo chown -R vmail:dovecot /etc/dovecot

Change the permissions on the `/etc/dovecot/` directory to be recursively read, write, and execute for the owner of the directory:

    sudo chmod -R o-rwx /etc/dovecot

Edit the service settings file `/etc/dovecot/conf.d/10-master.conf`.
Disable unencrypted IMAP and POP3 by setting the protocols’ `ports` to `0`. Uncomment the `port` and `ssl` variables:

    ...
    service imap-login {
        inet_listener imap {
            port = 0
        }
        inet_listener imaps {
            port = 993
            ssl = yes
        }
    ...
    }
    ...
    service pop3-login {
        inet_listener pop3 {
            port = 0
        }
        inet_listener pop3s {
            port = 995
            ssl = yes
        }
    }
    ...

Find the `service lmtp` section of the file and use the configuration shown below:

    ...
    service lmtp {
        unix_listener /var/spool/postfix/private/dovecot-lmtp {
            mode = 0600
            user = postfix
            group = postfix
        }
    ...
    }

Locate `service auth` and configure it as shown below:

    ...
    service auth {
    ...
        unix_listener /var/spool/postfix/private/auth {
            mode = 0660
            user = postfix
            group = postfix
        }

        unix_listener auth-userdb {
            mode = 0600
            user = vmail
        }
    ...
        user = dovecot
    }
    ...

In the `service auth-worker` section, uncomment the `user` line and set it to `vmail`:

    ...
    service auth-worker {
    ...
        user = vmail
    }

Edit `/etc/dovecot/conf.d/10-ssl.conf` file to require SSL and to add the location of your domain’s SSL certificate and key:

    ...
    # SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
    ssl = required
    ...
    ssl_cert = </etc/letsencrypt/live/mail.mydomain.com/fullchain.pem
    ssl_key = </etc/letsencrypt/live/mail.mydomain.com/privkey.pem

And restart all affected services:

    sudo systemctl restart postfix
    sudo systemctl restart dovecot

You should now be able to send and receive e-mail using properly configured e-mail client. If something is not working, trace the problem using system logs (mainly `/var/log/syslog`).

### 2. RBL checks

Nowadays, there are plenty of RBLs, but only few of them is trustworthy and stable enough so we can rely on them. We will connect to some of those lists to check whether e-mail should be processed or rejected.

First modify section `smtpd_recipient_restrictions` in your Postfix `main.cf` so it will look like this:

    smtpd_recipient_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_non_fqdn_recipient,
        reject_unknown_recipient_domain,
        reject_unlisted_recipient,
        reject_unauth_destination,
        reject_unauth_pipelining,
        reject_rbl_client bl.spamcop.net,
        reject_rbl_client b.barracudacentral.org,
        reject_rbl_client dnsbl.sorbs.net

Now just restart Postfix:

    sudo systemctl restart postfix

Whenever new e-mail arrives, you server will now ask three RBL if sernder of the e-mail is blacklisted.

### 3. Keywords blacklist

Create table for keywords blacklist:

    CREATE TABLE `mailblacklistedkeywords` (
    `id` smallint(5) unsigned NOT NULL AUTO_INCREMENT,
    `type` char(1) DEFAULT NULL,
    `pattern` varchar(255) DEFAULT NULL,
    `action` varchar(255) DEFAULT NULL,
    `description` varchar(255) DEFAULT NULL,
    `active` tinyint(1) unsigned NOT NULL DEFAULT '1',
    `amount_used` bigint(20) unsigned NOT NULL DEFAULT '0',
    `last_used` datetime DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `type` (`type`),
    KEY `pattern` (`pattern`),
    KEY `action` (`action`),
    KEY `active` (`active`),
    KEY `amount_used` (`amount_used`),
    KEY `last_used` (`last_used`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8;

The columns in this table have following purpose:

- `type` determines whether specieid phrase should be searched in e-mail header (`h`) or e-mail body (`b`)
- `patter` is the blacklisted phrase - you can user regular expressions here
- `action` specifies what Postfix should do in case the phrase is matched
- `active` allows to simply enable / disable blacklist entries

Populate table with test data:

    INSERT INTO `mailblacklistedkeywords`
    (`type`, `pattern`, `action`, `description`)
    VALUES
    ('b', 'blacklistedkeywordinbody', 'REJECT You are not welcome here', 'this is just a test keyword');

Now create function that will return filter action based on given keyword. This function also updates use amount of found filter and last used datetime at the same time.

    DELIMITER ;;
    CREATE FUNCTION `blacklisted_keyword_verify`(`filter_type` char(1), `filter_pattern` text) RETURNS varchar(255) CHARSET utf8
        DETERMINISTIC
    BEGIN
        DECLARE filter_id SMALLINT(5) DEFAULT 0;
        DECLARE filter_action VARCHAR(255) DEFAULT NULL;

        SELECT id, CONCAT(action, ' #', id) INTO filter_id, filter_action FROM mailblacklistedkeywords WHERE type = filter_type AND active = 1 AND filter_pattern REGEXP pattern LIMIT 1;

        UPDATE mailblacklistedkeywords SET amount_used = amount_used + 1, last_used = NOW() WHERE id = filter_id;

        RETURN filter_action;
    END;;
    DELIMITER ;

Add following lines to `/etc/postfix/main.cf`:

    header_checks = mysql:/etc/postfix/mysql-virtual-header-checks.cf
    body_checks = mysql:/etc/postfix/mysql-virtual-body-checks.cf

Create the file `/etc/postfix/mysql-virtual-header-checks.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT blacklisted_keyword_verify('h', '%s')

Create the file `/etc/postfix/mysql-virtual-body-checks.cf` with following content:

    user = mailuser
    password = mailpassword
    hosts = 127.0.0.1
    dbname = mailserver
    query = SELECT blacklisted_keyword_verify('b', '%s')

Restrict the permissions of the `/etc/postfix` directory to allow only its owner and the corresponding group:

    sudo chmod -R o-rwx /etc/postfix

And restart Postfix:

    sudo systemctl restart postfix

Now, if you send e-mail with phrase `blacklistedkeywordinbody` in body to your server, or try to send such e-mail from this server, you should get error response. Note that the error message generated by server contains Id of matched blacklist entry - this is very useful when you accidentaly blacklist something and you need to debug it.
