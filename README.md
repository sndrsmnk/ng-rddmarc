# ng-rddmarc

DMARC aggregate report to DB parser tool.
&copy; 2016, GPLv2, Sander Smeenk, <github@freshdot.net>

This tool will help with automatic parsing of DMARC RUA 'Aggregate Reports' to a MySQL database.
It can monitor a Maildir, for instance, and process reports that come in on the fly.


## Notes
This script can monitor a directory for changes to files. It does this by
utilising the <code>Filesys::Notify::Simple</code> Perl module. This module
will, by itself, monitor a directory by performing directory scans
periodically. You can get more efficient monitor methods by installing the
matching implementation for your OS:

OS      | Module
------- | -----------------------
Linux   | Linux::Inotify2
OSX     | Mac::FSEvents
BSD*    | Filesys::Notify::KQueue
Windows | Win32::ChangeNofity


## Invocation
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

## Consideration
Store the four 'conf:' lines with correct values in
<code>.ng-rddmarc.conf</code> and place it in your <code>$HOME</code> to
configure the database connection. If you want, you can use just
<code>-N</code> on the commandline, and have the other values filled in from
<code>.ng-rddmarc.conf</code>, but if <code>.ng-rddmarc.conf</code> also
specifies <code>$opt_N</code>, it will override the value from the commandline.

Default startup behaviour is to findall files within directories specified on
the command line to process. Use the <code>-n</code> option to prevent this
script from scanning for files within directories on startup.

If one or more file(s) are specified as arguments they are parsed as
<code>message/rfc822</code>-formatted 'email files' by default. Use the
<code>-x</code> option to indicate you're feeding in the actual XML-formatted
report files.

If one or more directories are specified as argument, it is assumed that every
normal file within that directory is a valid
<code>message/rfc822</code>-formatted 'email file', or XML-formatted report
file when the <code>-x</code> option was used.

Using the <code>-m</code> option enables detection of <code>Maildir</code>
structures in directories specified. This could be useful for monitoring your
own <code>Maildir/.dmarc.rua/<code> mailbox. On startup, meaning, every file in
the directory will be parsed _unless <code>-n</code> is used_ . Directories
within this directory will not be recursed into.

Database schema:
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



# Copyrights
&copy; 2016, GPLv2, Sander Smeenk <github@freshdot.net>
Some parts of this script are literal copies of, or are based on, work by:

Copyright 2012-2013, Taughannock Networks. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

Redistributions of source code must retain the above copyright
notice, this list of conditions and the following disclaimer.

Redistributions in binary form must reproduce the above copyright
notice, this list of conditions and the following disclaimer in the
documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY
WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
