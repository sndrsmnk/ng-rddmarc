# ng-rddmarc


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

You'll also need the following Perl modules installed:

Perl Module             | Ubuntu / Debian Package
----------------------- | -----------------------
Filesys::Notify::Simple | libfilesys-notify-simple-perl
MIME::Parser            | libmime-tools-perl
PerlIO::gzip            | libperlio-gzip-perl
Getopt::Std             | perl-modules
MIME::Words             | libmime-tools-perl
XML::Simple             | libxml-simple-perl
DBD::mysql              | libdbd-mysql-perl
DBI                     | libdbi-perl

And the '<code>gzip</code>' (usually default) and '<code>unzip</code>' (usually
non-default) packages/binaries. Make sure your version of <code>unzip</code> supports
the <code>-p</code> option to make unzip <code>extract files to pipe, no messages</code>.

Or adjust the code accordingly to your unzip binary. ;)


# Invocation
```
ng-rddmarc [opts] <filenames ...>
ng-rddmarc [opts] <directories ...>

Options:
    -h       show this text
    -s       show MySQL create table statements

    -d nn    print debug info, nn = level,
             -1 = all, 10 = sensible, 11 = xml dumps

    -r       replace existing reports in database

    -w       block and monitor for new files in path(s) to process
    -n       do not initially scan files in path(s) specified
    -m       detect Maildir: process files and monitor in new/ dir,
             move completed to cur/ dir.
    -c       if path is Maildir, also process cur/ on startup

    -C xx/yy Use file 'xx/yy' as configuration file.
             Not specifying the -C option will try these in order:
                $HOME/.ng-rddmarc.conf
                /etc/ng-rddmarc/ng-rddmarc.conf

  The following 'conf:' lines can be stored in a configuration file:
    -U xxx   database user        conf: $opt_U = "username";
    -P xxx   database password    conf: $opt_P = "password";
    -N xxx   database name        conf: $opt_N = "dbname";
    -H xxx   database hostname    conf: $opt_H = "hostname";
    -p       enable persistent database connection
             defaults to connect for each report processed (0)
                                  conf: $opt_p = 1;
```


# Configuration
See the <code>contrib/dot&#x5f;ng-rddmarc.conf</code> file for example
config. Place it somewhere the script expects it (see <code>--help</code>)
or use the <code>-C</code> parameter to point to your file and configure
the database connection. Configuration options specified via command
line override those specified in the configuration file.<br/>
A valid database environment is needed for operation.


# Consideration
Default startup behaviour is to process all files specified on the
command line and those found within directories specified on the
command line. There will be no recursion into subdirectories.
Use the <code>-n</code> option to prevent this script from scanning for
files within directories on startup. It will then only process files
explicitly specified on the command line.


# Filetypes
The tool will try to distinguish between XML formatted report files, with
the first five bytes <code>&lt;?xml</code>, and assume all other files are
MIME compatible <code>message/rfc822</code>-formatted 'email files', as
commonly found in Maildir structures, where the XML-report is a properly
formatted base64 encoded attachment. 


# Maildir
When a Maildir/ structure is specified, and <code>-m</code> is used, by
default only the <code>new/</code> directory is scanned for files to
process. The files will be moved to <code>cur/</code> after successful
processing. If you also want to scan and process all files in
<code>cur/</code> on startup, you can use the <code>-c</code>
option. Using <code>-w</code> with <code>-m</code> will cause the
<code>new/</code> directory to be monitored for events, even if you
specified the parent directory on the commandline.


# Block and wait
Use the <code>-w</code> flag in combination with at least one directory
on the command line to enable 'waiting for new events' in said
directory. You could run <code>ng-rddmarc</code> as a daemon and process
messages automatically as it 'watches' your (<code>-m</code>)Maildir/ for
new files to process and move to <code>cur/</code> (marking it 'Old
Mail' in your mail client).

Other non-Maildir/ directories may be mixed with Maildir/ type
directories and will be processed and watched as normal directories.


# Verbosity
The <code>-d</code> flag enables verbose output. There is no real logic
to the verbose levels at this moment. A value of <code>10</code> shows
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
mysql> FLUSH PRIVILEGES;
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
) charset=utf8;

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
) charset=utf8;
```


# Examples
We all love examples.

Considering a valid configuration file exists...

```shell
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/new
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/cur
$ ng-rddmarc -m /home/user/Maildir/.site.dmarc-rua/
```
All of these do the same, they scan the messages in <code>new/</code>
and move them to <code>cur/</code> since <code>-m</code> was used and
this was in fact a Maildir/-structure.

```shell
$ ng-rddmarc -w -m /home/user/Maildir/.site.dmarc-rua/
```
This will process all files in <code>new/</code> and move them to
<code>cur/</code> when finished. It will then wait for new files to
appear in the <code>new/</code> directory and process them, indefinitely.
You could run this from init/upstart/systemd as a 'service'.

```shell
$ ng-rddmarc -r ./report.xml
```
This processes the <code>./report.xml</code> file, replacing
records in the database if any exists, and quits.

```shell
$ ng-rddmarc /opt/dump/xmls/
```
Processes all files in <code>/opt/dump/xmls/</code>, skips
reports that are already in the database, and quits.

```shell
$ ng-rddmarc -w /opt/dir ./messagefile.eml
```
Processes all files in <code>/opt/dir</code> <i>and</i>
<code>./messagefile.eml</code>, after this it will wait for new files in
<code>/opt/dir</code> indefinitely.

... etc.


# Copyrights
&copy; 2016, GPLv2, Sander Smeenk <github@freshdot.net><br/>
Some parts of this script are literal copies of, or are roughly based
on, work by Taughannock Networks, licensed as follows:

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
