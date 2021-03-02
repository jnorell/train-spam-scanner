# train-spam-scanner

train-spam-scanner is a bash script to handle spamassassin or rspamd training from imap folders on a dovecot server.

The design tries to balance the pro's and con's of having users vs. mail administrators classify messages to train the scanner.

## Status

This script has initial functionality, and will be refactored/improved over time.

Currently running on Debian 10 calling spamassassin via amavisd-new or using rspamd.  It's fairly configurable.  It is being developed for use on an ISPConfig mail server, and the defaults are set accordingly.

## Requirements

train-spam-scanner requires dovecot version 2 and a working [SpamAssassin](README.spamassassin.md) or [rspamd](README.rspamd.md) setup.  

train-spam-scanner utilizes shared folders, so you must have dovecot's acl and imap_acl plugins enabled; these are set in the /etc/dovecot/dovecont.conf file for ISPConfig, eg:

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

# When creating any namespaces, you must also have a private namespace:  (not needed for ISPConfig)
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

You must have dovecot's iterate_query set as well.

Install the 'bindfs' and 'fdupes' packages.

## Design Considerations

For the best results, you must train a spam scanner from manually classified mail (ie. spam and ham), sorted by competent users/admins.  In practice accurately sorting is hard to do even on one's own mail at times, as you sometimes don't know if a given message is spam or something you actually gave permission to receive in the past but forgot about.  Sorting another user's mail accurately is impossible to do in many cases.  So you need users you can trust to classify accurately, and at times just toss out messages that you/they are not certain of.

On the other hand, you need a fair bit of both ham and spam to train a scanner well.  Neither SpamAssassin nor rspamd will even utilize bayes without 200 messages of each; ideally much more than that should be provided.  So how to get that much classified mail without an administrator/trusted user having to sort it all?  Harness the power/resources of all users, of course.  And with that, all the inaccuracies they will bring.  Don't want that valid newsletter any more?  Why bother unsubscribing, when you can just move it to the spam folder?

The design here tries to compromise.  We allow users to supply sorted ham/spam messages for training, but then we allow administrative/trusted users to moderate those messages and override users' decisions.  You can allow all users to supply training messages (train-spam-scanner -F), or only a select few if you only create training folders for those few mail accounts.

## Design

Use for a user is simple, each user will have a 'Train as Spam' and a 'Train as Non-Spam' folder, and messages put in either one will be trained accordingly (unless overridden by an admin).  When a mistake is made in classification, just pull the message out of the training folder and it'll get fixed at the next rebuild.

Use for an admin is only slightly more involved.  An admin will see two folders of 'incoming' mail to be classified, 'Admin.User Trained Spam' and 'Admin.User Trained Non-Spam'; these are the combination of all the users' training folders, and they are rebuilt frequently.  There are also 3 training folders into which email admins move messages from incoming, 'Admin.Spam', 'Admin.Non-Spam', and 'Admin.Ignore'.  To classify a message, the admin just moves a message from one of the incoming folders to a training folder.

Messages in the 'Admin.Spam' or 'Admin.Non-Spam' training folders will override the same messages that users have specified; this can be used to correct misclassifications by users.  Any messages found in the 'Admin.Ignore' folder will not be trained by the scanner, and this is where an admin should put messages that really cannot be classified accurately or with confidence.

## Implementation

The system is based on IMAP folders so any imap client, including the server's webmail interface, can be used with it.  The user training folders are just normal IMAP folders with a known name; the admin incoming and training folders are public folders (meaning they are 'shared' folders managed by the sysadmin; they are not available to all users) under the 'Admin.' namespace.

Rather than creating a new system user to own these public folders we use the existing 'vmail' user (which already has access to all mail); don't set a password for the vmail user, it is not needed and should not have one.  Admins must be other imap users on the same system, and are granted access to the training folders by the train-spam-scanner script.

Messages in users' training folders should be expired after some time (default is 6 months).  Messages in admin training folders should expire, probably later (default is 1 year); ideally retaining a minimum number of ham/spam irregardless of age, but that is not implemented.

sa-learn will be run as the amavis user (rspamc as the \_rspamc user), using a bindfs mount to map vmail->amavis (or vmail->\_rspamc) user id's to simplify permissions (and will mount only the public folder paths, not the entire /var/vmail).


## Considerations

You may want to exclude users' training folders from quota limits (to do so, alter iterate_query).

Users should be aware that any mail they put into the training folders will probably be reviewed/seen by an admin, and may be stored on the server for quite some time -- so don't put any sensitive information in there.  Also make sure that is legal in your country.

Users can remove a message from a training folder and it will not get added in the next round of training.  However, if an admin has already copied one of those messages to an override (aka training) folder, it will stay there, the user cannot remove it.  This may be changed in the future, when implementing 'incremental' bayes training mode (which must track all messages in admin training folders between training runs).  For now, such messages will be gone once expired (eg. after 1 year).

## Support

Use the [issue tracker](https://github.com/jnorell/train-spam-scanner/issues) for issues and suggestions, and send pull requests with changes.  I'll try to post a howto on howtoforge.com which allows some questions/discussion in comments.

