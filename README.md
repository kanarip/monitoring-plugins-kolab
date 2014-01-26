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
