Title: Fix a corrupt Firefox profile
Slug: firefox-corrupt
Date: 2021-04-16 20:25
Modified: 2021-04-16 20:25
Authors: Mark Mulligan
Category: SysAdmin
Tags: firefox, rsync
Summary: How to find a corrupt file with rsync and remove it so Firefox can start.

### <span style="color:red">**DISCLAIMER:**</span> This should not be done unless you have backups of your data and are prepared for data loss!

# Intro

My desktop has an old hard disk drive (HDD) that's starting to fail.  That's not a problem though, I have all my files backed up.  However, I would prefer to run this HDD until it's unuseable before I replace it.  I even bought a pair of solid states drives (SSD) in anticipation of needing to do a replacement in the short term.  While that hasn't been neccessary yet I have been experiencing the occasional corrupt file in my Firefox `.mozilla` directory.  Today was one of those days.

# Troubleshooting
My first step was to figure out which file was corrupted.  For that I decideid to use `rsync` to copy each file to a temporary directory and see which one(s) would cause the rsync to hang, indicating the file had been corrupted:

    :::shell
    mkdir /tmp/ff
    rsync -hravu --progress .mozilla/ /tmp/ff/

In hindsight, I probably could have added `--dry-run` to my rsync command and not bothered writing the files as simply reading them would have failed.

The command ran until it abruptly stopped on a file, in this case `places.sqlite`:

    :::shell
    $ rsync -hravu --progress .mozilla/ /tmp/ff/
    sending incremental file list
    ./
    firefox.last-version
                  3 100%    0.00kB/s    0:00:00 (xfr#1, ir-chk=1003/1005)
    extensions/
    firefox/

    ...

    firefox/6zflj7ny.default/old_cookies.sqlite
            524.29K 100%    2.13MB/s    0:00:00 (xfr#53, ir-chk=1101/1161)
    firefox/6zflj7ny.default/permissions.sqlite
            196.61K 100%  796.68kB/s    0:00:00 (xfr#54, ir-chk=1100/1161)
    firefox/6zflj7ny.default/pkcs11.txt
                876 100%    3.45kB/s    0:00:00 (xfr#55, ir-chk=1099/1161)
    firefox/6zflj7ny.default/places.sqlite
    ^Crsync: [receiver] write error: Broken pipe (32)
    rsync error: received SIGINT, SIGTERM, or SIGHUP (code 20) at rsync.c(644) [sender=3.1.3]
    rsync: [sender] write error: Broken pipe (32)

I hit `Ctrl + C` to cancel the rsync command (see the `^C` above) having found the troublesome file.

# Fixing

I removed the corrupt file:

    :::shell
    rm .mozilla/firefox/6zflj7ny.default/places.sqlite

and reran the rsync command:

    :::shell
    $ rsync -hravu --progress .mozilla/ /tmp/ff/
    sending incremental file list
    firefox/6zflj7ny.default/
    firefox/6zflj7ny.default/.parentlock
                  0 100%    0.00kB/s    0:00:00 (xfr#1, ir-chk=1150/1161)
    firefox/6zflj7ny.default/addonStartup.json.lz4
              3.02K 100%    0.00kB/s    0:00:00 (xfr#2, ir-chk=1143/1161)

    ...
          
    firefox/6zflj7ny.default/weave/failed/passwords.json
                 10 100%    0.49kB/s    0:00:00 (xfr#20, to-chk=123/1830)
    firefox/6zflj7ny.default/weave/failed/prefs.json
                 10 100%    0.49kB/s    0:00:00 (xfr#21, to-chk=116/1830)
    firefox/6zflj7ny.default/weave/failed/tabs.json
                 10 100%    0.49kB/s    0:00:00 (xfr#22, to-chk=109/1830)

    sent 2.49M bytes  received 907 bytes  4.99M bytes/sec
    total size is 269.88M  speedup is 108.19 (DRY RUN)

The command completed without issue and I was able to launch firefox once again.  For completeness, I verified the file was recreated with:

    ls .mozilla/firefox/6zflj7ny.default/places.sqlite

which returned a file with a timestamp later that when I removed the file previously:

    $ ls -la .mozilla/firefox/6zflj7ny.default/places.sqlite
    -rw-r--r-- 1 muxync muxync 10485760 Apr 16 20:53 .mozilla/firefox/6zflj7ny.default/places.sqlite

and removed the temporary directory:

    rm -r /tmp/ff/

# Conclusion

Of course this isn't something you should typically do.  If your HDD is crashing you should aim to backup your data and replace the drive as soon as possible.  I happen to be prepared for my HDD to stop working so now I'm just running on borrowed time.
