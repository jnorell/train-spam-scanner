# train-spam-scanner with SpamAssassin

train-spam-scanner is a bash script to handle spamassassin or rspamd training from imap folders on a dovecot server.  This file covers use with spamassassin.

## Status

train-spam-scanner had initial functionality developed using spamassassin and has been used with debian 8, 9 and 10.

It is working with bayes in sql and berkeley db; redis should work with the same configuration as sql, but has not been tested.

The time required to run sa-learn (particularly with txrep enabled) can be problematic with a large number of training messages, though a little sql server tuning can help quite a bit.  Allegedly storing bayes in redis should be much faster (though I don't know if txrep can also store in redis).

Testing with ~6400 training messages (where most messages had previously been trained, a typical daily run once implemented) ran about 450 messages/minute after mysql server tuning.

    Message handling time:  3.5 min.
    Training (sa-learn):    10 min. (with txrep + bayes)
                            --------
    Total time:             14 min.
    # training messages:    ~6400
    Overall speed:          ~450 messages per minute

Using train-spam-scanner together with training using dovecot's imap_sieve (eg. https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/) works well.  Training happens quickly when users move mail into the training folders, then any misclassification can be corrected when train-spam-scanner runs.  See the [README.rspamd.md](README.rspamd) for an exmaple working setup.

Todo:  to help improve sa-learn speed, the 'incremental' bayes training mode needs to be implemented, which should help tremendously (see notes within the script itself).

## Requirements

It requires a working spamassassin setup using a shared bayes database, and dovecot version 2.  If TxRep is configured it will be trained by sa-learn along with bayes.

Example Bayes and TxRep config from /etc/spamassassin/local.cf:

```
# Bayes configuration
bayes_store_module             Mail::SpamAssassin::BayesStore::MySQL
bayes_sql_dsn                  DBI:mysql:spamassassin:localhost
bayes_sql_username             spamassassin
bayes_sql_password             some_random_password
bayes_sql_override_username    amavis

bayes_expiry_max_db_size       5000000
bayes_auto_expire              0
bayes_auto_learn               0
bayes_learn_during_report      0
bayes_learn_to_journal         1
bayes_token_sources            all

# might need to work on these:
bayes_ignore_header            X-Bogosity
bayes_ignore_header            X-Spam-Flag
bayes_ignore_header            X-Spam-Status

# remember to load TxRep in v341.pre and create the database table
use_txrep                      1
txrep_factory                  Mail::SpamAssassin::SQLBasedAddrList
user_awl_dsn                   DBI:mysql:spamassassin:localhost
user_awl_sql_username          spamassassin
user_awl_sql_password          some_random_password
user_awl_sql_table             txrep
txrep_ipv4_mask_len            32
```

train-spam-scanner utilizes shared folders, so must have dovecot's acl and imap_acl plugins enabled, as well as dovecot's iterate_query set.  Dovecot acl and imap_acl plugins are set in the /etc/dovecot/dovecont.conf file for ISPConfig, eg:

```
cat <<'EOF' >>/etc/dovecot/dovecot.conf
# acl and imap_acl used for spam scanner training
mail_plugins = acl

protocol imap {
  mail_plugins = $mail_plugins acl imap_acl
}
plugin {
  acl = vfile
}

## When creating any namespaces, you must also have a private namespace:  (not needed for ISPConfig)
#namespace {
#  type = private
#  separator = .
#  prefix =
#  # location defaults to mail_location.
#  inbox = yes
#}
# Admin namespace for spam training.
namespace {
    type = public
    separator = .
    prefix = Admin.
    location = maildir:/var/vmail/admin/
    subscriptions = yes
}
EOF
```

Install the 'bindfs' and 'fdupes' packages.

Rather than creating a new system user to own these public folders we use the existing 'vmail' user (which already has access to all mail); don't set a password for the vmail user, it is not needed and should not have one.  Admins must be other imap users on the same system, and are granted access to the training folders by the train-spam-scanner script.

sa-learn will be run as the amavis user, using a bindfs mount to map vmail->amavis user id's to simplify permissions (and will mount only the public folder paths, not the entire /var/vmail).


## Considerations

Users can remove a message from a training folder and it will not get added in the next round of training.  However, if an admin has already copied one of those messages to an override (aka training) folder, it will stay there, the user cannot remove it.  This may be changed in the future, when implementing 'incremental' bayes training mode (which must track all messages in admin training folders between training runs).  For now, such messages will be gone once expired (eg. after 1 year).

