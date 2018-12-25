# 30-seconds-of-awk-code

## Welcome to 30 seconds of AWK code!

>An AWK implementation of 30-seconds-of-code.

**Note**:- This is in no way affiliated with the original [30-seconds-of-code](https://github.com/Chalarangelo/30-seconds-of-code/).

## Table of contents
 * [choosing right implementation]()
 * [basic I/O]()
 * [replacing grep]()
 * [replacing sed]()
 * [replacing head and tail]()
 * [performing math]()
 * [troubleshooting awk programs]()

## Choosing right implementation

There are three modern implementations to consider:

### GNU AWK known as `gawk`
 * best feature set
 * wide availability
 * moderate performance

### Mike Brennan's and Thomas E. Dickey's AWK known as `mawk`
 * best performance

### Busybox AWK known as `busybox awk`
 * smallest footprint
 * best availability (even on embedded linux)

### Lesson learned:
 * there are multiple AWK implementations
 * stick to the modern implementations `gawk`, `mawk` or `busybox awk`

## Basic I/O
AWK is able to read and write text streams very easily

```awk
function awk_showcase() {
  echo -e "AWK is still useful\ntext-processing  technology!" | \
    awk 'BEGIN{wcnt=0;print "lineno/#words/3rd-word:individual words\n" > "/dev/stderr"}
              {printf("% 6d/% 6d/% 8s:%s\n",NR,NF,$3,$0);wcnt+=NF}
           END{print "\nSummary:", NR, "lines/records,", wcnt, "words/fields" > "/dev/stderr"}'
}

$ awk_showcase
lineno/#words/3rd-word:individual words

     1/     4/   still:AWK is still useful
     2/     2/        :text-processing  technology!

Summary:2 lines/records, 6 words/fields

$ awk_showcase 2>/dev/null
     1/     4/   still:AWK is still useful
     2/     2/        :text-processing  technology!
```
### Lesson learned:
 * printing data can be done via `print` and `printf()` buildin functions
 * passing data to `stderr` stream is done redirecting to [awk special file handle `/dev/stderr`](https://www.gnu.org/software/gawk/manual/html_node/Special-FD.html)

## Replacing `grep`

```
# 0. text data generation
$ ps auxwww > /tmp/ps.log
```

### essential case `grep mypattern`

```awk
# 1. grep command
$ grep ^avahi /tmp/ps.log
avahi      671  0.0  0.0  28108  2820 ?        Ss   11:45   0:00 avahi-daemon: running [linux.local]
avahi      688  0.0  0.0  27980   228 ?        S    11:45   0:00 avahi-daemon: chroot helper

# 2. mimicking grep command, keeping awk code as short as possible (using default/implicit actions)
$ awk '/^avahi/' /tmp/ps.log
avahi      671  0.0  0.0  28108  2820 ?        Ss   11:45   0:00 avahi-daemon: running [linux.local]
avahi      688  0.0  0.0  27980   228 ?        S    11:45   0:00 avahi-daemon: chroot helper

# 3. mimicking grep command, awk code as readable as possible
$ awk '/^avahi/ {print $0}' /tmp/ps.log
avahi      671  0.0  0.0  28108  2820 ?        Ss   11:45   0:00 avahi-daemon: running [linux.local]
avahi      688  0.0  0.0  27980   228 ?        S    11:45   0:00 avahi-daemon: chroot helper
```


### basic example `grep -F mypattern`

```awk
# 1. let's have multiple `gvfs-udisks2-volume-monitor+` processes
$ grep -F "gvfs-udisks2-volume-monitor+" /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+
gdm       1630  0.0  0.1 413940  8420 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+
f_ii      2711  0.0  0.1 414288  8564 tty2     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+

# 2. we need to select the *first one* running under user `gdm` (silly & fragile solution relying on `gdm` one comes first)
$ grep -F "gvfs-udisks2-volume-monitor+" -m 1 /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+

# 3. we need to select the one running under user gdm (more correct solution)
$ grep -F "gvfs-udisks2-volume-monitor+" /tmp/ps.log | grep -E -m 1 "^gdm[ \t]+"
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+

# 4. naive implementation of the grep command 2. would be
$ awk 'index($0,"gvfs-udisks2-volume-monitor+") > 0 {print $0;exit(0);}' /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+

# 5. more correct implementation of the grep command 3. is
$ awk '$1 == "gdm" && index($0,"gvfs-udisks2-volume-monitor+") > 0 {print $0;exit(0);}' /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /usr/libexec/gvfs-udisks2-volume-monitor+
```

### multifile grepping `grep -E mypattern <file-glob>`

```awk
# we have multiple uptime files showinh uptime at 13:00, 14:00, 15:00 and 16:00.
$ ls uptime.*
uptime.13:00  uptime.14:00  uptime.15:00  uptime.16:00

# let's show files where there were at least 10 active users logged in
#   using grep with extended regexp
$ grep -E ' [1-9][0-9]+ users' uptime.*
uptime.14:00: 12:04:11 up  2:19, 10 users,  load average: 0.28, 0.32, 0.27
uptime.15:00: 13:04:13 up  2:19, 11 users,  load average: 0.26, 0.31, 0.27
uptime.16:00: 14:04:15 up  2:19, 17 users,  load average: 0.26, 0.31, 0.27

# do the same with awk more optimal way
$ awk '$4 >= 10 {print FILENAME ":" $0}' uptime.*
uptime.14:00: 12:04:11 up  2:19, 10 users,  load average: 0.28, 0.32, 0.27
uptime.15:00: 13:04:13 up  2:19, 11 users,  load average: 0.26, 0.31, 0.27
uptime.16:00: 14:04:15 up  2:19, 17 users,  load average: 0.26, 0.31, 0.27
```


### Lesson learned:
 * `awk` is text-processing swiss-knife and handles very easily everything what `grep` know
 * there are [AWK implicit/default actions which can make programs shorter but also much less readable](https://www.gnu.org/software/gawk/manual/html_node/Intro-Summary.html#Intro-Summary)
 * every word in input stream is accessible via `$i` variable
 * an implicit conversion to number takes place so we can easily compare words to integer or float
 * `FILENAME` variable keeps name of currently processed (read) file

## Replacing `sed`

TODO


## Replacing `head` and `tail`


TODO

## Performing math


TODO

## Troubleshooting awk programs

TODO




