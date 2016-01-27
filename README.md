# ng-rddmarc

[TOC]


DMARC aggregate report to DB parser tool.<br/>
&copy; 2016, GPLv2, Sander Smeenk, <github@freshdot.net>

This tool will help with automatic parsing of DMARC RUA 'Aggregate Reports' to a MySQL database.<br/>
It can monitor a Maildir, for instance, and process reports that come in on the fly.


# Notes
This script can monitor a directory for changes to files. It does this
by utilising the <code>Filesys::Notify::Simple</code> Perl module. This
module will, by itself, monitor a directory by performing directory
scans periodically. You can get more efficient monitor methods by
installing the matching implementation for your OS:

OS      | Module
------- | -----------------------
Linux   | Linux::Inotify2
OSX     | Mac::FSEvents
BSD     | Filesys::Notify::KQueue
Windows | Win32::ChangeNofity


# Invocation
```
perl script.pl [opts] <filenames ...>
perl script.pl [opts] <directories ...>

Options:
    -h       show this text
    -s       show MySQL create table statements
    -d nn    print debug info, nn = level, -1 = all

    -r       replace existing reports
    -x       read XML files rather than mail messages

    -w       block and monitor for new files in path(s) to process
    -n       do not initially scan files in path(s) specified
    -m       detect Maildir: process files and monitor in new/ dir,
                 move completed to cur/ dir.
    -c       if path is Maildir, also process cur/ on startup

    -U xxx   database user        conf: $opt_U = "username";
    -P xxx   database password    conf: $opt_P = "password";
    -N xxx   database name        conf: $opt_N = "dbname";
    -H xxx   database hostname    conf: $opt_H = "hostname";
```


# Consideration
See the <code>contrib/dot&#x5f;ng-rddmarc.conf</code> file for example
config. Place it in <code>$HOME/.ng-rddmarc.conf</code> to configure
the database connection. If not all database options are specified on
the command line, the script will look for this file and 'merge' that
with the command line specified values. This means, for example, you
could use just <code>-N</code> on the commandline, and have the other
values filled in from <code>.ng-rddmarc.conf</code>. <strong>Please
note:</strong> if <code>.ng-rddmarc.conf</code> also specifies
<code>$opt&#x5f;N</code>, it will override the value from the
commandline.

Other <code>$opt&#x5f;</code> values can also be set from
<code>.ng-rddmarc.conf</code>. For example, setting <code>$opt&#x5f;w =
1;</code> will enable -w by default. <strong>Please note:</strong>
specifying all of <code>-U</code>, <code>-P</code>, <code>-N</code> and
<code>-H</code> on the command line disables loading of
<code>.ng-rddmarc.conf</code>.

Default startup behaviour is to find all files within directories
specified on the command line to process. There will be no recursion
into subdirectories of paths specified. Use the <code>-n</code> option
to prevent this script from scanning for files within directories on
startup.

When a Maildir/ structure is specified, and <code>-m</code> is used, by
default only the <code>new/</code> directory is scanned for files to
process. They will be moved to <code>cur/</code> after successful
processing. If you also want to scan and process all files in
<code>cur/</code> on startup too, you can use the <code>-c</code>
option. Using <code>-w</code> with <code>-m</code> will cause the
<code>new/</code> directory to be monitored for events, even if you
specified the parent directory on the commandline.

Other non-Maildir/ directories may be mixed with Maildir/ type
directories and will be processed and watched as normal directories.

All files found in directories or specified on the command line are
parsed as <code>message/rfc822</code>-formatted 'email files' by
default. Use the <code>-x</code> option to indicate you're feeding in
the actual XML-formatted report files.

Use the <code>-w</code> flag in combination with at least one directory
on the command line to enable 'waiting for new events' in said
directory. You could run <code>ng-rddmarc</code> as a daemon and process
messages automatically as it 'watches' your (<code>-m</code>)Maildir for
new files to process and move to <code>cur/</code> (marking it 'Old
Mail' in your mail client).

The <code>-d</code> flag enables debugging. There is no real logic to
the debug levels at this moment. A value of <code>10</code> shows
everything but dumps of the parsed XML-bodies and is most useful if
something is going awry.


# Database

This script was developed with a MySQL database as backend. But it uses
Perl DBI and it should thus be trivial to change to a different database
backend. The MySQL create statements can be obtained by running
<code>ng-rddmarc -s</code>. See this simplistic example on how to create
the database:

```shell
$ ssh root@databaseserver
$ mysql
mysql> CREATE DATABASE dmarc DEFAULT CHARACTER SET utf8;
mysql> GRANT ALL ON dmarc.* TO 'dmarc'@'%' IDENTIFIED BY 'yourplainpassword';
mysql> exit
Bye
$ ng-rddmarc -s | mysql -u dmarc -p -h databaseserver dmarc
```


## Database schema:
```mysql
CREATE TABLE report (
  serial int(10) unsigned NOT NULL AUTO_INCREMENT,
  mindate timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  maxdate timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  domain varchar(255) NOT NULL,
  org varchar(255) NOT NULL,
  reportid varchar(255) NOT NULL,
  PRIMARY KEY (serial),
  UNIQUE KEY domain (domain,reportid)
) charset=utf8 ENGINE=MyISAM;

CREATE TABLE rptrecord (
  serial int(10) unsigned NOT NULL,
  ip int(10) unsigned,
  ip6 binary(16),
  rcount int(10) unsigned NOT NULL,
  disposition enum('none','quarantine','reject'),
  reason varchar(255),
  dkimdomain varchar(255),
  dkimresult enum('none','pass','fail','neutral','policy','temperror','permerror'),
  spfdomain varchar(255),
  spfresult enum('none','neutral','pass','fail','softfail','temperror','permerror'),
  KEY serial (serial,ip),
  KEY serial6 (serial,ip6)
) charset=utf8 ENGINE=MyISAM;

CREATE TABLE failure (
  serial int(10) unsigned NOT NULL AUTO_INCREMENT,
  org varchar(255) NOT NULL, -- reported-domain
  bouncedomain varchar(255), -- MAIL FROM bouncebox@bouncedomain
  bouncebox varchar(255),
  fromdomain varchar(255), -- From: frombox@fromdomain
  frombox varchar(255),
  arrival TIMESTAMP,
  sourceip int unsigned, -- inet_aton(source-ip)
  sourceip6 BINARY(16), -- inet_6top(source-ip)
  headers TEXT,
  PRIMARY KEY(serial),
  KEY(sourceip),
  KEY(fromdomain),
  KEY(bouncedomain)
) charset=utf8 ENGINE=MyISAM;
```

## Examples
We all love examples.

Considering a valid <code>$HOME/.ng-rddmarc.conf</code> exists...

```shell
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/new
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/cur
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/
```
All of these do the same, they scan the messages in new/, since
<code>-m</code> was used and this was in fact a Maildir/-structure.

```shell
$ ng-rddmarc -w -m /home/user/Maildir/.site.dmarc-rua/
```
This will process all files in <code>new/</code> and move them to
<code>cur/</code> when finished. It will then wait for new files to
appear in the <code>new/</code> directory and process them, indefinitely.
You could run this from init/upstart/systemd as a 'service'.

```shell
$ ng-rddmarc -r -x ./report.xml
```
This processes the <code>./report.xml</code> file as (<code>-x</code>)XML,
replacing records in the database if any exists, and quits.

```shell
$ ng-rddmarc -x /opt/dump/xmls/
```
Processes all files in path as XML, skips reports that are already in
the database, and quits.

```shell
$ ng-rddmarc -w /opt/dir ./messagefile.eml
```
Processes all files in <code>/opt/dir</code> <i>and</i>
<code>./messagefile.eml</code> as default <code>message/rfc822</code>
formatted files, after this it will wait for new files in /opt/dir
indefinitely.

... etc.


# Copyrights
&copy; 2016, GPLv2, Sander Smeenk <github@freshdot.net><br/>
Some parts of this script are literal copies of, or are based on,
work by Taughannock Networks, licensed as follows:

> Copyright 2012-2013, Taughannock Networks. All rights reserved.
> 
> Redistribution and use in source and binary forms, with or without
> modification, are permitted provided that the following conditions
> are met:
> 
> Redistributions of source code must retain the above copyright
> notice, this list of conditions and the following disclaimer.
> 
> Redistributions in binary form must reproduce the above copyright
> notice, this list of conditions and the following disclaimer in the
> documentation and/or other materials provided with the distribution.
> 
> THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
> "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
> LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
> A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
> HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
> INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
> BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
> OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
> AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
> LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY
> WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
> POSSIBILITY OF SUCH DAMAGE.
