---
title: "how to deal with LKML = offlineimap + mutt"
date: 2018-01-09T22:12:00+05:30
tags: ["mutt", "offlineimap", "lkml"]
slug: "mailing-lists"
---

[LKML](http://vger.kernel.org/lkml/) is a _very_ high volume mailing list. I
accumulate around 500 emails during the time I spend sleeping. It's a kitchen
sink for every kernel patch and discussion that is going on, which is a good
thing and a bad thing. Because I care about only a few of them. Most users
subscribe to and read only subsystem specific mailing lists. The trick is to run
scripts on top of them to find the emails of interest.

I had a specific need where I wanted to find emails where the sysfs ABI was
being modified.

How did I go about this?

#### Step 1

It's easy to run custom scripts of varying complexity if your mails are
available locally. I first setup [**offlineimap**](http://www.offlineimap.org/) to
download and keep all my mails offline. It's never a bad idea to keep a backup.
This is the minimal config needed to get started :

~/.offlineimaprc

```zsh
[general]
# list of accounts, I have only one
accounts = aishpant

[Account aishpant]
# identifier for the local repository
localrepository = local-aishpant
# identifier for the remote repository
remoterepository = remote-aishpant
# download emails starting from this date
maxage = 2017-12-25

# local repository settings
[Repository local-aishpant]
type = Maildir
# where should the emails go?
# this local folder needs to exist
localfolders = /home/a/mail/aishpant

# remote repository settings
[Repository remote-aishpant]
type = Gmail
# Gmail specific username and password
remoteuser = xxx@gmail.com
remotepass = xxxpass
sslcacertfile = /etc/ssl/certs/ca-certificates.crt
# synchronise only these remote folders
# I have three folders of interest
folderfilter = lambda folder: folder in ['INBOX', 'LKML', 'attrDoc']
```
Then you can just run offlineimap to start fetching all your emails.

#### Step 2

Set-up [**mutt**](http://www.mutt.org/) to read and send emails. I use mutt anyway
to send patches as the kernel community accepts only text based emails.

~/.muttrc

```zsh
## for sending mails
set realname = "Aishwarya Pant"
set from =  "aishpant@gmail.com"
set smtp_url = "smtps://xxx@gmail.com@smtp.gmail.com:465/"
set smtp_pass = "xxxpass"
set sendmail="/usr/bin/esmtp"

## appearance
set sort = 'threads'
set sort_aux = 'reverse-last-date-received'
set charset = "utf-8"

## for reading emails, read from local
# needs to be consistent with offlineimap
set mbox_type = Maildir
set folder = /home/a/mail/aishpant
# folder in which to start mutt
set spoolfile = +/INBOX/
# cache for even faster
set header_cache = /home/a/mail/cache/

## writing mail
set editor = "vim"
```

#### Step 3

If you run offlineimap, it will sync emails only once. After that you will need
to set-up a cron job or daemon to keep running the service in background. Do not
use the offlineimap autorefresh to sync emails as it takes up a lot of memory.
Memory usage on my system was ~800 Mb after it had been running for an hour. I
used the [**systemd** user](https://wiki.archlinux.org/index.php/Systemd/User)
timer to periodically sync emails.

First, create two files - `offlineimap.service` and `offlineimap.timer`

```zsh
## offlineimap.service
[Unit]
Description=Offlineimap Service (oneshot)
Documentation=man:offlineimap(1)

[Service]
# execute the action and stop
Type=oneshot
ExecStart=/usr/bin/offlineimap -o -u basic
# give 120 seconds for offlineimap to gracefully stop before hard killing it
TimeoutStopSec=120

[Install]
WantedBy=mail.target
```

```zsh
## offlineimap.timer
[Unit]
Description=Offlineimap Query Timer

[Timer]
# start the service 2 minutes after a boot-up
OnBootSec=2m
# the timer will run the service every 5 minutes
OnUnitInactiveSec=5m

[Install]
WantedBy=default.target
```

Put both files in the folder `/etc/systemd/user`. Now, to enable the service and
the timer run :

```zsh
$ systemctl --user enable offlineimap.service offlineimap.timer
$ systemctl --user start offlineimap-oneshot.service
```
The service needs to be started once before the timer can kick in.

You can check the status of the service using the `systemctl` command. (The
`--user` flag has to be passed every time.)

```zsh
$ systemctl --user list-timers
NEXT                         LEFT          LAST                         PASSED       UNIT                      A
Wed 2018-01-10 14:06:50 IST  3min 50s left Wed 2018-01-10 14:01:24 IST  1min 35s ago offlineimap-oneshot.timer o

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```
```bash
$ systemctl --user status offlineimap-oneshot
● offlineimap-oneshot.service - Offlineimap Service (oneshot)
   Loaded: loaded (/etc/systemd/user/offlineimap-oneshot.service; enabled; vendor preset: enabled)
   Active: inactive (dead) since Wed 2018-01-10 14:01:50 IST; 1min 1s ago
     Docs: man:offlineimap(1)
  Process: 6552 ExecStart=/usr/bin/offlineimap -o -u basic (code=exited, status=0/SUCCESS)
 Main PID: 6552 (code=exited, status=0/SUCCESS)

Jan 10 14:01:27 mordor offlineimap[6552]: Syncing INBOX: Gmail -> Maildir
Jan 10 14:01:27 mordor offlineimap[6552]: Syncing attrDoc: Gmail -> Maildir
Jan 10 14:01:30 mordor offlineimap[6552]: Deleting 2 messages (23726,23729) in Maildir[INBOX]
Jan 10 14:01:30 mordor offlineimap[6552]: Deleting 1 messages (23703) in Gmail[INBOX]
Jan 10 14:01:31 mordor offlineimap[6552]: Syncing LKML: Gmail -> Maildir
Jan 10 14:01:46 mordor offlineimap[6552]: Copy message UID 217124 (1/3) remote-aishpant:LKML -> local-aishpant
Jan 10 14:01:46 mordor offlineimap[6552]: Copy message UID 217125 (2/3) remote-aishpant:LKML -> local-aishpant
Jan 10 14:01:49 mordor offlineimap[6552]: Copy message UID 217126 (3/3) remote-aishpant:LKML -> local-aishpant
Jan 10 14:01:50 mordor offlineimap[6552]: *** Finished account 'aishpant' in 0:25
Jan 10 14:01:50 mordor systemd[585]: Started Offlineimap Service (oneshot).
```

#### Step 4

With all my set-up done, now I can write scripts that parse through the LKML
emails and put the emails of my interest in another folder (attrDoc in this
case).

The offlineimap mail directory contains three folders:

* 'tmp' for messages that are in process of being delivered
* 'new' for messages that have not been seen by any mail application such as mutt
  or the Gmail web client
* 'cur' for messages that have been viewed

I have a [python
script](https://github.com/aishpant/attribute-documentation/blob/master/parseEmail.py)
that looks for patch emails in the new messages of the LKML folder, and checks
if there are any sysfs attribute declaring macros like DEVICE\_ATTR in the diff.
If there are any, it will put these emails in the new message folder of attrDoc.
Now when I open mutt and navigate to the attrDoc folder, I see saner emails and
all this effort has paid off!

In conclusion, this seems like a lot of work for filtering emails. It _was_.
There might be simpler and effective ways for achieving the same result. But
this is what works for me currently. This is not recommended for every one. For
normal usage, I more than happy with the Gmail web client. I set-up all my
mailing list filters over there (i.e. procmail is overkill).

#### To-do's

* Modify the script to copy the entire message thread instead of just one email
  to the attrDoc folder
* Read passwords from [GNU Pass](https://www.passwordstore.org/)
* Configure auto-completion for contacts, colors and vim like navigation
  in mutt ( or [Neomutt](https://www.neomutt.org/) ?)

Here are some sources I used to figure this out :

* Offlineimap and mutt pages from the [Arch Linux
  wiki](https://wiki.archlinux.org/). The Arch Linux community is the best and
  the wiki has answers to every thing!
* Offlineimap oneshot systemd
  [config](https://github.com/OfflineIMAP/offlineimap/tree/master/contrib/systemd)
