Kolab Groupware Plugins for Monitoring
======================================

The plugins in this repository have been specifically authored for Kolab
Groupware, that may or may not be managed by a configuration management suite
such as Puppet.

Many of the Nagios plugins you might find on the web, for example, do not:

*   **accept `--extra-opts` parameters**,

    which in combination with a generated `plugins.ini` would allow you to
    distribute configuration across staging environments,

*   **monitor services end-to-end**,

    which in elastic, distributed environments (scaled horizontally) is a
    prerequisite in monitoring and reporting,

*   **simply don't exist**,

    Such as `check_saslauthd`.

Munin plugins you might find may not be appropriately trending performance --
some count files in `/var/lib/imap/proc/`, which may contain stale files,
misrepresenting the number of connections, authenticated connections and number
of unique users. This used to be the case for the `cyrus-imapd` Load graphing:

![Cyrus IMAP Discrete Murder Topology Frontend Load Graph](https://raw.github.com/kanarip/monitoring-plugins-kolab/master/munin/images/cyrus-imapd_murder_load-day.png)

Our solution to miscounting is rather simple, and our plugin uses
`lsof +d /var/lib/imap/proc/` to list proc files actually in use.

`check_email_delivery`
----------------------

Example for the *check_email_delivery* plugin:

    ./check_email_delivery \
        --plugin=./check_smtp_send \
        --plugin=./check_imap_receive \
        --hostname=localhost \
        --imap-username=john.doe@example.org \
        --imap-password=somepass \
        --mail-from=john.doe@example.org \
        --mail-to=john.doe@example.org \
        --mail-subject='Something Unique'

The plugin will submit a message with the corresponding ``To:``, ``From:`` and
``Subject:`` headers (using the ``localhost`` SMTP server, which in this example
has ``127.0.0.0/8`` in `postconf mynetworks` and
``permit_mynetworks`` in its ``smtpd_recipient_restrictions``), and check the
IMAP server (also at ``localhost``) for delivery, and if the message is found,
flag it as `\Seen`, `\Deleted` and EXPUNGE the folder.

A more complex example may be:

    ./check_email_delivery \
        --plugin=./check_smtp_send \
        --plugin=./check_imap_receive \
        --smtp-server=smtp.example.org \
        --smtp-username=jane.doe@example.org \
        --smtp-password=janespass \
        --smtp-port=587 \
        --smtp-starttls \
        --imap-server=imap.example.org \
        --imap-username=john.doe@example.org \
        --imap-password=johnspass \
        --imap-starttls \
        --imap-mailbox='Filtered Monitoring Messages' \
        --mail-from=john.doe@example.org \
        --mail-to=jane.doe@example.org \
        --mail-subject='Something Unique' \
        --mail-header='X-Nagios-Check: %TOKEN1%'

What this checks, end-to-end, is the following:

1)  Connection to submission,
2)  STARTTLS,
3)  Authentication against submission,
4)  The Kolab SMTP Access Policy,
5)  Relay on to the IMAP backend servers (over LMTP or SMTP),
6)  Mail delivery to mailboxes on the IMAP backend servers (usually over LMTP),
7)  Sieve filtering (optionally),
8)  IMAP frontend connectivity,
9)  IMAP proxy capability (what backend mailserver is this mailbox on),
10) Performance.

For environments that indeed do use staging, or have a shared Nagios set up for
an otherwise multi-tenant environment, it is useful to not have to specify each
of these plugin switches.

We have therefore added the use of `--extra-opts`, and the intended use is to
distribute different `/etc/nagios/plugins.ini` files to systems that require
such.

For example, a standalone development system might want to connect to the SMTP
and IMAP server using address `localhost`:

    [check_email_delivery]
    starttls=true
    username=john.doe@example.org
    **hostname=localhost**
    **password=devpass**
    (...)

while a production system (in a distributed environment) might need to connect
with the SMTP server on address `smtp.example.org`, and the IMAP server on
address `imap.example.org`:

    [check_email_delivery]
    starttls=true
    username=john.doe@example.org
    **smtp-server=smtp.example.org**
    **password=prodpass**
    **imap-server=imap.example.org**
    (...)

In a real-life scenario, which could be summarized as a distributed environment
with separate web servers, mail exchangers and IMAP backends, what we are
interested in is both smarhost relay delivery from our webservers (whom send out
notification emails to our users), and (authenticated) submission delivery such
as users perform it.

Checking the availability of each individual service is merely assisting a
system administrator in trying to determine **what** component of the end-to-end
functionality is misbehaving. To detect what functionality is impacted or lost,
the service is monitored end-to-end, and in a full mesh:

*   `$x` web servers, times
*   `$y` mail exchangers, times
*   `$z` IMAP backend servers, times
*   2 types of service checks (smarthost relay and submission).

Makes for 16 end-to-end service checks if you have 2 servers of each type, and
54 service checks if you have 3 of each type. Multiply using your own numbers
before assessing whether or not it is reasonable to lower the check frequency in
your environment.
