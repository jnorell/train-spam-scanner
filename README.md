# train-spam-scanner

TODO:  write this readme.  ;) This is very rough, real documentation will follow.

This script has initial functionality, and will be refactored/improved in time.  It requires
shared folders enabled (so eg. dovecot acl plugin); details will follow.


Bayes config from /etc/spamassassin/local.cf:

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
```

