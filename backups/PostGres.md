# Backup the Postgres Database with crontab

## Summary

What we just discussed is only marginally a Mastodon set of issues.

It is really a collection of ways to administrate the Linux (Unix)
system thar runs your instance.  I strongly encourage you to experiment
on a separate system with the commands mentioned so you are fluent
with their intent and behavior.

DO NOT just use these commands blindly as printed here, until you
know exactly what will happen on NON production version of the Instance.

In a word, practice on a test server UNLESS this is just old-hat.


## Terms


Let `MUSER` be equal to the username that holds your `./live` production
server.  By default this is `mastodon`.  Wherever you see `MUSER`, just
replace that with whatever your username is that runs your Mastodon 
instance.   


## As `root` you need to prepare

If you want to hold the backup files in a place outside
of the home directory of the user that runs the Mastodon instance
you'll need to select a place for these files.

Let us suppose that location is `/var/mastodon/backups/daily`

```
# mkdir -p /var/mastodon/backups/daily
```

We need to let `MUSER` be able to write to this directory tree.

```
# chown -R MUSER.MUSER /var/mastodon
```

## As `MUSER`

Become `MUSER`

Either:

```
# su - $MUSER
```

Or login as that user.  Your situation may be different.  In either case
become `MUSER` (default would be `mastodon`)

As `MUSER` select your preferred `EDITOR`

(Opt to make this part of your `~/.profile` so it's always set when you
are `MUSER`)

You can type this every time you login, or make it part of your
`~/.profile` so it's always set.  Your call.

```
$ export EDITOR=vi
```

Or whatever editor you prefer.  `nano`, etc..

So, the place has been established where we want to store the
Postgres backup files.

Let's construct a hint to Postgres to know how to access Postgres
without using a password on the command line.

As `MUSER` create this file:  `~/.pgpass`

Make the mode of the file `0600`

```
$ chmod 0600 ~/.pgpass
```

Edit the file to contain:
```
*:*:DB_NAME:DB_USER:PASSWORD
```

Your `DATABASE` name is the name of the database you selected
when you installed the Mastodon instance.  If you forgot it, you
can find it in `~/live/.env.production`

```
$ grep DB ~/live/.env.production
```

Note the `DB_NAME` and `DB_USER`  

If `DB_NAME` was `mastodon_prod` and `DB_USER` was `joe` it would look
like this, so far:


```
*:*:mastodon_prod:joe:PASSWORD
```

But the password... what about that?  

Well, the documentation for installing Mastodon overlooks this detail.

Back when you installed Mastodon, when it asked you to create the Postgres
user `mastodon` it did NOT specify how to create the Postgres user
*with a password*.  You kinda of need this, I think.

So it may come as a shock, but you need that password.   

I am not sure where in the `~/live` file tree the Mastodon server
software actually stores the password. If it does, then I'd love to know.

But I went this route --

When I installed Mastodon to begin with, I created the `mastodon` (aka
`MUSER`) password WITH the account on Postgres:

Goal, to run this Postgres query, pick a better password maybe.

`ALTER USER mastodon WITH PASSWORD 'pineapple';`


One solution:

```
# sudo -u postgres psql
psql (14.5 (Ubuntu 14.5-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# ALTER USER mastodon WITH PASSWORD 'pineapple';
postgres=# \q
# exit
```

So, with that password when the Mastodon instance was created you
use THAT password.. 

Back to the backup... Back to the `~/.pgpass`

If...  the database name was `mastodon_prod` and if the
user that you created is the default `mastodon` and if the password
was `pineapple`, then this is what `~/.pgpass` would look like.


```
*:*:mastodon_prod:mastodon:pineapple
```

You can test this:

```
$ psql 
```

Did you login to Postgres?

If not, check the steps above.

If you did, quit Postgres..


## Establishment of the `crontab` entry for backup


Now that we have

1. Established a password-less way to invoke `psql`
2. Established a directory location on the system to store the backup


We can setup the `crontab` entry to cause the backup to occur.


As `MUSER` (aka `mastodon` if that is what you used..)

```
$ crontab -e
```

Let's not be confused here.  `cron` is a daemon that runs on the
Unix (Linux) host.   `cron` will execute commands as defined by
the schedule parameters in each and every `crontab` file in the system.

Think of `crontab` as "the `tab`-le of entries to execute".  

Anyway ...

This will open a file (`-e` means to edit the `crontab` file for the current
user invoking `crontab`)

The format of `crontab` entries 

```
              field          allowed values
              -----          --------------
              minute         0–59
              hour           0–23
              day of month   1–31
              month          1–12 (or names, see below)
              day of week    0–7 (0 or 7 is Sun, or use names)
```

For example

```
1 2 3 4 * /usr/bin/barf
```

means that at 02:01 on the 3rd day of APRIL, the command `/usr/bin/barf`
will execute as the user `MUSER`

```
0 5 * * * /usr/bin/barf
```

means that at 05:00 any day, on any month regardless of day of
the week, it will run `/usr/bin/barf`


See also man page for `crontab`

```
$ man -s 5 crontab
```



```
0 0 * * * /usr/bin/barf
```

At 00:00, it will run /usr/bin/barf.


But that won't backup.. We need to change the command!

So let's revise the command to run to actually backup Postgres
for your Mastodon server instance.

So if you want your Postgres backup of your Mastodon server to happen
at midnight in the time zone of the server you use to run 
the Instance then use this:

```
* 0 * * * /usr/bin/pg_dump -w -U mastodon DB_NAME > "/var/mastodon/backups/daily/daily_$(date +\%Y\%m\%d\%H).psql"
```


Ok A lot going on here...

1.  At 00:00 it runs the command `pg_dump`  As a habit always provide the absolute path name to programs to let `cron` run.  If the `PATH` variable were to be setup so that `pg_dump` was not the intended `pg_dump` then you can cause
a program to execute that was not intended.  Just a thing to remember:
Always use absolute path names to programs to invoke via `cron`.

2. The flags to `pg_dump` are `-w` (don't use passwords on command line -- don't expect passwords to be given on the command line) (`man pg_dump` for explanation.)

3. The next argument is the `DB_NAME` (database name) to backup.  Replace
`DB_NAME` with the name of the database you used.  

4. The next argument redirects the output to a file.  The file is going
to be stored in the location we setup previously.  But we want the NAME
of the file to be unique for when the file is created.  What is next is
a shell-ism not something related to `pg_dump` at all.  In `bash` for
instance this syntax `$(cmd)` will replace with whatever the output is
of `cmd`.  So what we are getting here is a string of characters of the date 
in `YYYY-MM-DD-HH` format.  (Try it out, `$ date +%Y%m%d%H`  -- you don't need to escape the `%` signs on the command line, but you do need to escape them when the command is inside the quotes in the `crontab` entry.)

5. The whole path is in side doublequotes so that `cron` will use the result string inside the doublequotes as the filename to write the dump.  And this is also
why we had to escape the `%`.  If you know a more elegant way, use it.



Exit the editor.

Once you do exit the editor, the Linux (Unix) system will register
the new `crontab` and perform the duties therein at the appointed time
and date.  You don't need to fiddle with it (unless you want to change
the schedule -- or add other tasks).


## Speaking of Other Tasks

One thing you'll likely want to is write a small shell script that will
prune the files that are saved for backups.  Left unattended, that
`crontab` entry will just keep making backups every day, forever.


I'll show that process later.

But the nuts and bolts would look like this:


```
0 19 * * * /bin/rm -f `/usr/bin/find /var/mastodon/backups/daily -iname 'daily*.psql' -mtime +30`
```

Note the way the back-ticks and ticks go.

The meaning.

1.  Run `/bin/rm -f`  The `-f` is force and this will help if there are NO files to remove. It will fail silently if NO files match.
2.  What is going to be removed?  The files listed by the command inside the back-ticks. That key upper-left on your keyboard, not the single-quote.  That command to "find files that are in the directory named `daily*.psql` AND are older
than 30 days"  is `find /that/path -iname 'daily*.psql' -mtime +30`
3. So what we're doing is passing a file list to `/bin/rm -f` and that file list is made up of all files in that path that satisfy a) the name is `daily*.psql` and b) the file is older than 30 days.

Adjust your dates and so on to suit your needs.

Oh, and this runs at 19:00 every day.   So time the "cleanup" to not interfere with the creation of the backups themselves  -- just to even out the load
on the server -- the files won't ever be in conflict.  A file to cleanup 
will never be a newly made backup file if `-mtime +N` is sufficiently 
large.

