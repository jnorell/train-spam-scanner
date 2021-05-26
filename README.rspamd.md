# train-spam-scanner with Rspamd

train-spam-scanner is a bash script to handle spamassassin or rspamd training from imap folders on a dovecot server.  This file covers use with rspamd.

## Status

train-spam-scanner has been used with rspamd on Debian 10, and should working with any bayes store, but is so far only tested with bayes stored in redis.

Performance with rspamd training seems to be in another class altogether, not even close to how long it takes spamassassin to train.  I timed a train-spam-scanner run of rspamd vs spamassassin, with identical vm config/hardware/storage, and the rspamd system took 1min 59sec to train from 4608 messages (almost 100% messages already trained), where spamassassin took 14min 28sec to train from 4840 messages (again, almost 100% retraining).  Though note the spamassassin system stored bayes in mysql, not redis (I understand should be much faster if using redis), and also trained TxRep.

Using train-spam-scanner together with training using dovecot's imapsieve (eg. https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/) works well.  Training happens quickly when users move mail into the training folders, then any misclassification can be corrected when train-spam-scanner runs.

### Requirements

train-spam-scanner requires a working rspamd setup using with working bayes store, and dovecot version 2.  If fuzzy_check is configured it will not be trained along with bayes, but the script could be easily modified to do so.

train-spam-scanner utilizes shared folders, so must have dovecot's acl and imap_acl plugins enabled, as well as dovecot's iterate_query set.

Install the 'bindfs' and 'fdupes' packages.

Rather than creating a new system user to own these public folders we use the existing 'vmail' user (which already has access to all mail); don't set a password for the vmail user, it is not needed and should not have one.  Admins must be other imap users on the same system, and are granted access to the training folders by the train-spam-scanner script.

rspamc will be run as the \_rspamc user, using a bindfs mount to map vmail->\_rspamc user id's to simplify permissions (and will mount only the public folder paths, not the entire /var/vmail).


## Considerations

Rspamd currently has no ability to unlearn a message once trained.  If a message is trained as Spam, it can be retraied as Non-Spam, but if you as the admin are unsure of the categorization and want to use the Ignore folder, nothing will happen (the message will remain in the last trained state until it expires, eg. a year later).


## train-spam-scanner with rspamd setup on Debian 10

This is is a brief setup for use with rspamd on a Debian 10 box acting as an ISPConfig mail server; setup for other systems will be similar.  Note if using ISPConfig that any customizations made to dovecot or rspamd config files need to be made to any corresponding conf-custom templates, or they will be overwritten in future ISPConfig updates.  The config below uses the dovecot_custom.conf.master template introduced in 3.2.3.

Start with a working ISPConfig mail server with rspamd running.

Create a \_rspamc user to train as:

```
useradd --home-dir /var/run/rspamd --no-create-home --system _rspamc
passwd --lock _rspamc
```

Dovecot will be setup with imapsieve based training, needs the acl and imap_acl plugins enabled, and the 'Admin.' namespace created:

```
cat <<'EOF' >>/etc/dovecot/conf.d/99-ispconfig-custom-config.conf

# config for imapsieve based on https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/
plugin {
  sieve_plugins = sieve_imapsieve sieve_extprograms

  # From elsewhere to 'Train as Spam' folder
  imapsieve_mailbox1_name = 'Train as Spam'
  imapsieve_mailbox1_causes = COPY
  imapsieve_mailbox1_before = file:/usr/lib/dovecot/sieve/report-spam.sieve

  # From 'Train as Spam' folder to elsewhere
  imapsieve_mailbox2_name = *
  imapsieve_mailbox2_from = 'Train as Spam'
  imapsieve_mailbox2_causes = COPY
  imapsieve_mailbox2_before = file:/usr/lib/dovecot/sieve/report-ham.sieve

  # From elsewhere to 'Train as Non-Spam' folder
  imapsieve_mailbox3_name = 'Train as Non-Spam'
  imapsieve_mailbox3_causes = COPY
  imapsieve_mailbox3_before = file:/usr/lib/dovecot/sieve/report-ham.sieve

  sieve_pipe_bin_dir = /usr/lib/dovecot/sieve

  sieve_global_extensions = +vnd.dovecot.pipe +vnd.dovecot.environment
}

# acl and imap_acl used by train-spam-scanner
mail_plugins = $mail_plugins acl
protocol imap {
  mail_plugins = $mail_plugins acl imap_acl imap_sieve
}
plugin {
  acl = vfile
}

# namespace for train-spam-scanner admin training folders
namespace {
    type = public
    separator = .
    prefix = Admin.
    location = maildir:/var/vmail/admin/
    subscriptions = yes
}
EOF

cp /etc/dovecot/conf.d/99-ispconfig-custom-config.conf /usr/local/ispconfig/server/conf-custom/install/dovecot_custom.conf.master
```

In dovecot-sql.conf we will ignore quota for the training folders (and increase for Trash).

```
sed -i -e 's/^password_query /#password_query /g' /etc/dovecot/dovecot-sql.conf
sed -i -e 's/^user_query /#user_query /g' /etc/dovecot/dovecot-sql.conf

cat <<'EOF' >>/etc/dovecot/dovecot-sql.conf

# modified to allow more quota in Trash and ignore Spam Training folders
password_query = SELECT email as user, password, maildir as userdb_home, CONCAT( maildir_format, ':', maildir, '/', IF(maildir_format='maildir', 'Maildir', maildir_format)) as userdb_mail, uid as userdb_uid, gid as userdb_gid, CONCAT('*:storage=', quota, 'B') AS userdb_quota_rule, 'Trash:storage=+25%%' as userdb_quota_rule2, 'Deleted Messages:storage=+25%%' as userdb_quota_rule3, 'Deleted Items:storage=+25%%' as userdb_quota_rule4, 'Train as Spam:ignore' AS userdb_quota_rule5, 'Train as Non-Spam:ignore' AS userdb_quota_rule6, CONCAT(maildir, '/.sieve') as userdb_sieve FROM mail_user WHERE (login = '%u' OR email = '%u') AND `disable%Ls` = 'n' AND server_id = 'server_id_replacement' AND NOT EXISTS (SELECT domain_id FROM mail_domain WHERE domain = '%d' AND active = 'n' AND server_id = server_id_replacement)

user_query = SELECT email as user, maildir as home, CONCAT( maildir_format, ':', maildir, '/', IF(maildir_format='maildir','Maildir',maildir_format)) as mail, uid, gid, CONCAT('*:storage=', quota, 'B') AS userdb_quota_rule, 'Trash:storage=+25%%' as userdb_quota_rule2, 'Deleted Messages:storage=+25%%' as userdb_quota_rule3, 'Deleted Items:storage=+25%%' as userdb_quota_rule4, 'Train as Spam:ignore' AS userdb_quota_rule5, 'Train as Non-Spam:ignore' AS userdb_quota_rule6, CONCAT(maildir, '/.sieve') as sieve FROM mail_user WHERE (login = '%u' OR email = '%u') AND `disable%Ls` = 'n' AND server_id = 'server_id_replacement' 
EOF

server_id=$(grep server_id /usr/local/ispconfig/server/lib/config.inc.php | cut -d"'" -f4)
sed -i -e "s/server_id_replacement/${server_id}/g" /etc/dovecot/dovecot-sql.conf
```

Now create the report-ham.sieve and report-spam.sieve sieve scripts, and the learn-ham.rspamd.sh and learn-spam.rspamd.sh shell scripts they call.

```
mkdir /usr/lib/dovecot/sieve/

cat <<'EOF' >/usr/lib/dovecot/sieve/report-ham.sieve
# imapsieve spam training
# based on https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/
# and https://github.com/darix/dovecot-sieve-antispam-rspamd/

# report-ham.sieve runs both when moving mail into 'Train as Non-Spam' as well as
# when moving mail out of 'Train as Spam' folders

require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.mailbox" "*" {
  set "mailbox" "${1}";
}

# this is needed for deleting mail in 'Train as Spam' (usually copies to Trash)
if string :matches "${mailbox}" ["*.Trash", "Trash", "Junk", "*.Deleted Items", "Deleted Items", "*.Deleted Messages", "Deleted Messages"] {
  stop;
}

# ignore train-spam-scanner training folders
# (not needed for doveadm, but safeguard for mail clients moving mail there)
if string :matches "${mailbox}" "Admin.*" {
  stop;
}

if environment :matches "imap.user" "*" {
  set "username" "${1}";
}

pipe :copy "learn-ham.rspamd.sh" [ "${username}" ];

EOF


cat <<'EOF' >/usr/lib/dovecot/sieve/report-spam.sieve
# imapsieve spam training
# based on https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/
# and https://github.com/darix/dovecot-sieve-antispam-rspamd/

# report-spam.sieve runs when moving mail into 'Train as Spam' folders

require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables", "imap4flags"];

if environment :matches "imap.user" "*" {
  set "username" "${1}";
}

addflag "\\Seen";

pipe :copy "learn-spam.rspamd.sh" [ "${username}" ];

EOF


cat <<'EOF' >/usr/lib/dovecot/sieve/learn-spam.rspamd.sh
#!/bin/bash

# imapsieve spam training
# based on https://doc.dovecot.org/configuration_manual/howto/antispam_with_sieve/
# and https://github.com/darix/dovecot-sieve-antispam-rspamd/

action=$(basename "$0" .rspamd.sh)
action=${action#learn-}

case "${action}" in
ham|spam) ;;
*)
        logger -p mail.err "Can't figure out action (ham/spam) from script name '$0'"
        exit 1
        ;;
esac

# get "imap.user" (from sieve environment), also available as $USER
deliver_to=$USER

# defaults
RSPAMD_CONTROLLER_PASSWORD=/etc/dovecot/rspamd-controller.password
RSPAMD_CONTROLLER_SOCKET=
RSPAMD_CONTROLLER_HOST=
RSPAMD_CLASSIFIER=

if [ -r "/etc/dovecot/rspamd-controller.conf.sh" ]; then
        source "/etc/dovecot/rspamd-controller.conf.sh"
fi

# RSPAMD_CONTROLLER_PASSWORD is supposed to be a filename, pointing to
# a file including the password.
# If it contains a password directly, store it in ${password} instead.
password=
if [ "${RSPAMD_CONTROLLER_PASSWORD:0:1}" != '.' -a "${RSPAMD_CONTROLLER_PASSWORD:0:1}" != '/' ]; then
        logger -p mail.debug "Plaintext password given (should use a password file instead)"

        password=${RSPAMD_CONTROLLER_PASSWORD}
        RSPAMD_CONTROLLER_PASSWORD=
elif [ ! -r "${RSPAMD_CONTROLLER_PASSWORD}" ]; then
        logger -p mail.err "Missing password file for rspamc"
        exit 1
fi

if [ -z "${RSPAMD_CONTROLLER_HOST}" ] ; then
        logger -p mail.debug "Using socket connection to rspamd"

        args=()
        if [ -n "${RSPAMD_CONTROLLER_SOCKET}" ]; then
                args+=(-h "${RSPAMD_CONTROLLER_SOCKET}")
        fi
        if [ -n "${RSPAMD_CONTROLLER_PASSWORD}" ]; then
                args+=(-P "${RSPAMD_CONTROLLER_PASSWORD}")
        fi
        if [ -n "${RSPAMD_CLASSIFIER}" ]; then
                args+=(-c "${RSPAMD_CLASSIFIER}")
        fi

        if [ -n "${password}" ]; then
                # pass password through pipe; printf is a builtin and won't be
                # visible in ps; the generated path should look like /dev/fd/...
                # and always start with a '/'
                exec /usr/bin/rspamc -P <(printf "%s\n" "${password}") "${args[@]}" -d "${deliver_to}" "learn_${action}"
        else
                exec /usr/bin/rspamc "${args[@]}" -d "${deliver_to}" "learn_${action}"
        fi
else
        logger -p mail.debug "Using HTTP connection to rspamd"

        headers=(
                -H "Deliver-To: ${deliver_to}"
        )
        if [ -n "${RSPAMD_CONTROLLER_PASSWORD}" ]; then
                # curl supports files for headers, but needs the full header
                # generating it on the fly below, but read password here.
                read password < "${RSPAMD_CONTROLLER_PASSWORD}"
        fi
        if [ -n "${RSPAMD_CLASSIFIER}" ]; then
                headers+=(-H "Category: ${RSPAMD_CLASSIFIER}")
        fi

        # pass password header through pipe; printf is a builtin and won't
        # be visible in ps
        exec /usr/bin/curl \
                --silent \
                -H @<(printf "password: %s\r\n" "${password}") \
                "${headers[@]}" \
                --data-binary @- \
                "${RSPAMD_CONTROLLER_HOST}/learn${action}"
fi
EOF

cat <<'EOF' > /etc/dovecot/rspamd-controller.conf.sh
# Path to file containing the controller password
# (Or, if it doesn't start with '/' or '.', the password itself.
# But it might leak the password through ps to other users)
# Note:  Get this from the ISPConfig Server Config for this mail server
RSPAMD_CONTROLLER_PASSWORD=/etc/dovecot/rspamd-controller.password
# passed to rspamc with the -h option (host and port)
RSPAMD_CONTROLLER_SOCKET=
# if set uses curl instead of rspamc; should start with http: or https:
RSPAMD_CONTROLLER_HOST=
# classifier to learn for (default by rspamc: bayes), e.g. `bayes_user`
RSPAMD_CLASSIFIER=
EOF


sievec /usr/lib/dovecot/sieve/report-ham.sieve
sievec /usr/lib/dovecot/sieve/report-spam.sieve
chmod +x /usr/lib/dovecot/sieve/learn-spam.rspamd.sh
ln -s /usr/lib/dovecot/sieve/learn-spam.rspamd.sh /usr/lib/dovecot/sieve/learn-ham.rspamd.sh
chown root:dovecot /etc/dovecot/rspamd-controller.conf.sh
chmod 640 /etc/dovecot/rspamd-controller.conf.sh
```

Now we configure the rspamd controller password, which in ISPConfig you can look up from `System > Server Config > {server} > Mail`:


```
touch /etc/dovecot/rspamd-controller.password
chown root:dovecot /etc/dovecot/rspamd-controller.password
chmod 640 /etc/dovecot/rspamd-controller.password
echo 'your-rspamd-controller-password-here' > /etc/dovecot/rspamd-controller.password
```

Install the 'bindfs' and 'fdupes' packages:

```
apt install bindfs fdupes
```

Now download and configure train-spam-scanner itself.  You specify valid imap accounts for ADMIN_USERS:

```
wget -O /usr/local/sbin/train-spam-scanner https://raw.githubusercontent.com/jnorell/train-spam-scanner/master/train-spam-scanner
chmod +x /usr/local/sbin/train-spam-scanner

mkdir /etc/train-spam-scanner
wget -O /etc/train-spam-scanner/train-spam-scanner.conf https://raw.githubusercontent.com/jnorell/train-spam-scanner/master/train-spam-scanner.conf

sed -i 's/^SCANNER=/#SCANNER=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^ADMIN_USERS=/#ADMIN_USERS=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^TRAIN_AS_USER=/#TRAIN_AS_USER=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^TRAIN_AS_GROUP=/#TRAIN_AS_GROUP=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^TRAIN_BIND_MNT=/#TRAIN_BIND_MNT=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^DATA_DIR=/#DATA_DIR=/g' /etc/train-spam-scanner/train-spam-scanner.conf
sed -i 's/^RSPAMD_CONTROLLER_PASSWORD=/#RSPAMD_CONTROLLER_PASSWORD=/g' /etc/train-spam-scanner/train-spam-scanner.conf

cat <<'EOF' >>/etc/train-spam-scanner/train-spam-scanner.conf
# Local settings
SCANNER="rspamd"
ADMIN_USERS=( "admin1@domain.tld" "admin2@domain.tld" )
TRAIN_AS_USER='_rspamc'
TRAIN_AS_GROUP='_rspamc'
TRAIN_BIND_MNT='/var/run/rspamd/mnt'
DATA_DIR='/var/lib/train-spam-scanner'
RSPAMD_CONTROLLER_PASSWORD=/etc/train-spam-scanner/rspamd-controller.password
EOF

cp /etc/dovecot/rspamd-controller.password /etc/train-spam-scanner/rspamd-controller.password
chown root:_rspamc /etc/train-spam-scanner/rspamd-controller.password
chmod 640 /etc/train-spam-scanner/rspamd-controller.password
mkdir /var/lib/train-spam-scanner

cat <<'EOF' >/etc/cron.d/train-spam-scanner 
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# bring user trained messages into admin training folders
# (does not train the spam scanner)
35 * * * *  root  /usr/local/sbin/train-spam-scanner -F -M none

# train the spam scanner with sorted mail
15 */3 * * *  root  /usr/local/sbin/train-spam-scanner -f -C
EOF
```

And last, run train-spam-scanner to create the admin and user training folders:

```
train-spam-scanner -f -F -M none
```

