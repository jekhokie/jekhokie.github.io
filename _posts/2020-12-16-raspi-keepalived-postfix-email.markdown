---
layout: post
title:  "Email Alerts for keepalived Failover"
date:   2020-12-16 21:37:00 -0400
categories: keepalived postfix monitoring
logo: postfix.jpg
---

In a [previous post]({% post_url 2020-12-14-unifi-pihole-ad-blocking-with-failover %}) setting up a Pi-hole failover pair of Raspberry Pi instances, it was
mentioned that an improvement might be to add monitoring for when a failover occurs so you can be alerted and correct the situation. While Monit and other
monitoring-related tools are possible to use, there is a much simpler approach by using the native email capability in keepalived that notifies email
addresses when the instance state changes. This article details how to set up [postfix](http://www.postfix.org/) capability and configure keepalived to
email a set of pre-defined addresses using an existing Gmail email account.

## Disclaimer

While this tutorial covers a process that works well to alert if both the `MASTER` and `BACKUP` instances are running and a failover occurs, it does not in
fact cover a scenario where a `BACKUP` is down and a failover is attempted - in that case, no notification would be sent as the processes would both be dead
and never send an email about state changes that occur because the failover itself would not occur/would fail.

## Prerequisites

This tutorial assumes that you've completed [this previous tutorial]({% post_url 2020-12-14-unifi-pihole-ad-blocking-with-failover %}) and have a working
pair of Pi-hole instances (or any other pair of components using keepalived to maintain a failover functionality).

Additionally, you will need a valid/working `@gmail.com` Gmail account - while this is not the only way to configure email alerts (you can use almost any
other email account/type), this tutorial is opinionated and will use this account type for the configurations defined.

A pre-step for the below tutorial is to obtain an Application Password for your Gmail account - navigate to your Gmail `myaccount` (profile) and under
the `Security` section, add an `Application Password` that will use `Mail` capabilities and leave the device as `Other`. Once presented with the Application
Password, copy it for safekeeping as we'll use it as part of the Postfix configuration.

## Postfix Installation

**Note**: Steps here were mostly taken from [this post](https://medium.com/swlh/setting-up-gmail-and-other-email-on-a-raspberry-pi-6f7e3ad3d0e) - no claim
of ownership over this material.

On each of the primary/secondary instances of Raspberry Pi instances where keepalived is installed and running, let's first install Postfix:

```bash
$ sudo apt-get -y install postfix
$ libsasl2-modules
```

When prompted via menus, select that this will be an `Internet Site`, and provide a unique name for the `System mail name` (something akin to a hostname
or similar, so long as it's unique). Finally, ensure Postfix is enabled to start on boot:

```bash
$ sudo systemctl enable postfix.service
```

## Postfix Configuration

**Note**: Security hardening configurations in this section were mostly taken from [this post](https://linux-audit.com/postfix-hardening-guide-for-security-and-privacy/).
No claim of ownership is made.

Let's get the password configured for Postfix - create the file `/etc/postfix/sasl/sasl_password` with the following contents, making sure to replace
`<USERNAME>` and `<PASSWORD>` with your Gmail username and Application Password copied from the Prerequisites section above, respectively:

```
[smtp.gmail.com]:587 <USERNAME>@gmail.com:<PASSWORD>
```

Next, create a hash file for Postfix to use this information, which will create a file `sasl_password.db`:

`sudo postmap /etc/postfix/sasl/sasl_password`

Secure both of these files as they obviously contain sensitive information:

```bash
$ sudo chmod 600 /etc/postfix/sasl/sasl_password
$ sudo chmod 600 /etc/postfix/sasl/sasl_password.db
```

Finally, we'll configure Postfix - edit the file `/etc/postfix/main.cf` and ensure the following configuration settings are present and set:

```
disable_vrfy_command = yes
inet_interfaces = loopback-only
mynetworks = "127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128"
smtpd_helo_required = yes

relayhost = [smtp.gmail.com]:587

smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_tls_security_level = encrypt
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_password
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

Then, restart the Postfix process:

```bash
$ sudo systemctl restart postfix.service
```

## keepalived Configuration

Now that Postfix is running on each Raspberry Pi instance where keepalived is running, let's configure keepalived to use the local host for
email forwarding when state changes occur. Edit the configuration file `/etc/keepalived/keepalived.conf` on each of the primary/secondary hosts to
include the following content in addition to the existing content, ensuring you specify an email address you have access to for the `<EMAIL_NOTIFY>`
parameter. Note that the hostnames and email addresses used reflect hostnames related to Pi-hole instances, as that is where these configurations
originated for this tutorial:

```
# pihole-master.localdomain
global_defs {
    notification_email {
        <EMAIL_NOTIFY>
    }

    notification_email_from root@pihole-master.localdomain

    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id pihole-master.localdomain
}

vrrp_instance ADBLOCK {
    ...
    smtp_alert
    ...
}
```

```
# pihole-backup.localdomain
global_defs {
    notification_email {
        <EMAIL_NOTIFY>
    }

    notification_email_from root@pihole-backup.localdomain

    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id pihole-backup.localdomain
}

vrrp_instance ADBLOCK {
    ...
    smtp_alert
    ...
}
```

Once the above configurations are in place, restart the keepalived service on each of the primary/secondary instances:

```bash
$ sudo systemctl restart keepalived.service
```

## Testing It Out!

Now that you have everything configured, let's perform a test. On the primary keepalived instance, shut down keepalived,
wait a few minutes, and then restart keepalived:

```bash
$ sudo systemctl start keepalived.service
# wait 2-3 minutes
$ sudo systemctl start keepalived.service
```

If you check the notify email we configured earlier, you should see several emails show up indicating the `MASTER` and `BACKUP`
switch states/roles twice - once to fail over during the keepalived shutdown, and second when the `MASTER` returned to its
primary state - things are working well now!

## Troubleshooting

If things didn't exactly go as planned, it's possible your Postfix configurations are not correct. Try to troubleshoot by
sending emails from the command line like so, replacing `<NOTIFY_EMAIL>` with the email where you expect messages to be sent.
Investigate the logs in `/var/log/messages` once the below command is issued to see what might be going wrong with the
configuration:

```bash
$ echo "test" | mail -s "test" <NOTIFY_EMAIL>
```

## Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [Setting Up Gmail and Other Emails...](https://medium.com/swlh/setting-up-gmail-and-other-email-on-a-raspberry-pi-6f7e3ad3d0e)
- [Postfix Hardening Guide...](https://linux-audit.com/postfix-hardening-guide-for-security-and-privacy/)
