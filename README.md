# train-spam-scanner

train-spam-scanner is a bash script to handle spamassassin training from imap folders on a dovecot server.

The design tries to balance the pro's and con's of having users vs. mail administrators train spamassassin.

## Status

This script has initial functionality, and will be refactored/improved over time.

Currently running on Debian 8 calling spamassassin via amavisd-new, it's fairly configurable via variables.  It is being developed for use on an ISPConfig mail server, and the defaults are set accordingly.

It is working with bayes in berkeley db; it may need enhanced for sql or redis (not yet tested).

Testing with ~3500 training messages, the message handling is tolerably fast but sa-learn is not, taking about 1 second per message.  Tests with bayes in sql and redis need done, which may help.  Also need to create an 'incremental' bayes training mode, which should help tremendously (see notes within the script itself).

## Requirements

It requires a working spamassassin setup using a shared bayes database, and dovecot version 2.  If TxRep is configured it will be trained by sa-learn along with bayes.

Example Bayes and TxRep config from /etc/spamassassin/local.cf:

```
# Bayes configuration
bayes_expiry_max_db_size 5000000
bayes_auto_expire 0
bayes_auto_learn 0
bayes_learn_during_report 0
bayes_learn_to_journal 1
bayes_token_sources all

# might need to work on these:
bayes_ignore_header X-Bogosity
bayes_ignore_header X-Spam-Flag
bayes_ignore_header X-Spam-Status

# remember to load TxRep in v341.pre and create the database table
use_txrep 1
txrep_factory Mail::SpamAssassin::SQLBasedAddrList
user_awl_dsn DBI:mysql:spamassassin:127.0.0.1
user_awl_sql_username spamassassin
user_awl_sql_password some_random_password
user_awl_sql_table txrep
```

It utilizes shared folders, so must have dovecot's acl and imap_acl plugins enabled, as well as dovecot's iterate_query set.

Install the 'bindfs' and 'fdupes' packages.


## Design Considerations

For the best results, you must train spamassassin from manually classified mail (ie. spam and ham), sorted by competent users/admins.  In practice accurately sorting is hard to do even on one's own mail at times, as you sometimes don't know if a given message is spam or something you actually gave permission to receive in the past but forgot about.  Sorting another user's mail accurately is impossible to do in most cases.  So you need users you can trust to classify accurately, and at times just toss out messages that you're not certain of.

On the other hand, you need a fair bit of both ham and spam to train a scanner well.  Spamassassin won't even utilize bayes without 200 messages of each; ideally much more than that should be provided.  So how to get that much classified mail without an administrator/trusted user having to sort it all?  Harness the power/resources of all users, of course.  And with that, all the inaccuracies they will bring.  Don't want that valid newsletter any more?  Why bother unsubscribing, when you can just move it to the spam folder?

The design here tries to compromise.  We allow users to supply sorted ham/spam messages for training, but then we allow administrative/trusted users to moderate those messages and override users' decisions.

## Design

Use for a user is simple, each user will have a 'Train as Spam' and a 'Train as Non-Spam' folder, and messages put in either one will be trained accordingly (unless overridden by an admin).  When a mistake is made in classification, just pull the message out of the training folder and it'll get fixed at the next rebuild.

Use for an admin is only slightly more involved.  An admin will see two folders of 'incoming' mail to be classified, 'Admin/User Trained Spam' and 'Admin/User Trained Non-Spam'; these are the combination of all the users' training folders, and they are rebuilt frequently.  There are also 3 training folders into which email admins move messages from incoming, 'Admin/Spam', 'Admin/Non-Spam', and 'Admin/Ignore'.  To classify a message, the admin just moves a message from one of the incoming folders to a training folder.

Messages in the 'Admin/Spam' or 'Admin/Non-Spam' training folders will override the same messages that users have specified; this can be used to correct misclassifications by users.  Any messages found in the 'Admin/Ignore' folder will not be trained by the scanner, and this is where an admin should put messages that really cannot be classified accurately or with confidence.

## Implementation

The system is based on IMAP folders so any imap client, including the server's webmail interface, can be used with it.  The user training folders are just normal IMAP folders with a known name; the admin incoming and training folders are public folders (meaning they are 'shared' folders managed by the sysadmin; they are not available to all users) under the 'Admin/' namespace.

Rather than creating a new system user to own these public folders we use the existing 'vmail' user (which already has access to all mail); don't set a password for the vmail user.  Admins must be other imap users on the same system, and are granted access to the training folders by the train-spam-scanner script.

Need to load the dovecot acl and imap_acl plugins; these are set in the /etc/dovecot/dovecont.conf file for ISPConfig, eg:

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

# When creating any namespaces, you must also have a private namespace:
namespace {
  type = private
  separator = /
  prefix =
  # location defaults to mail_location.
  inbox = yes
}
# Admin namespace for spam training.
namespace {
    type = public
    separator = /
    prefix = Admin/
    location = maildir:/var/vmail/admin/
    subscriptions = yes
}
EOF
```

Need dovecot iterate_query enabled, which is in /etc/dovecot/dovecot-sql.conf if using ISPConfig.

Messages in users' training folders should be expired after some time (default is 6 months).  Messages in admin training folders should expire, probably later (default is 1 year); ideally retain a minimum number of ham/spam irregardless of age, but that is not implemented.

sa-learn will be run as the amavis user, using a bindfs mount to map vmail->amavis user id's to simplify permissions (mount only the public folder paths, not the entire /var/vmail).


## Considerations

You may want to exclude users' training folders from quota limits (to do so, alter iterate_query).

Users should be aware that any mail they put into the training folders will probably be reviewed/seen by an admin, and may be stored on the server for quite some time -- so don't put any sensitive information in there.  Also make sure that is legal in your country.

Users can remove a message from a training folder and it will not get added in the next round of training.  However, if an admin has already copied one of those messages to an override (aka training) folder, it will stay there, the user cannot remove it.


