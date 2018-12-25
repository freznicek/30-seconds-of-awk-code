# 30-seconds-of-awk-code

## Welcome to 30 seconds of AWK code!

>An AWK implementation of 30-seconds-of-code.

**Note**:- This is in no way affiliated with the original [30-seconds-of-code](https://github.com/Chalarangelo/30-seconds-of-code/).

## Table of contents
 * [choosing right implementation](#choosing-right-implementation)
 * [basic I/O](#basic-io)
 * [replacing grep](#replacing-grep)
 * [replacing sed](#replacing-sed)
 * [replacing head and tail](#replacing-head-and-tail)
 * [performing math](#performing-math)
 * [prettyprinting long awk programs](#prettyprinting-long-awk-programs)
 * [troubleshooting awk programs](#troubleshooting-awk-programs)

## Choosing right implementation

There are three modern implementations to consider:

### GNU AWK known as `gawk`
 * best feature set
 * wide availability
 * moderate performance

### Mike Brennan's and Thomas E. Dickey's AWK known as `mawk`
 * best performance ([example](https://brenocon.com/blog/2009/09/dont-mawk-awk-the-fastest-and-most-elegant-big-data-munging-language/))

### Busybox AWK known as `busybox awk`
 * smallest footprint
 * best availability (even on embedded linux)

### Lesson learned:
 * there are multiple AWK implementations
 * stick to the modern implementations `gawk`, `mawk` eventually `busybox awk`

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
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+
gdm       1630  0.0  0.1 413940  8420 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+
f_ii      2711  0.0  0.1 414288  8564 tty2     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+

# 2. we need to select the *first one* running under user `gdm` (silly & fragile solution relying on `gdm` one comes first)
$ grep -F "gvfs-udisks2-volume-monitor+" -m 1 /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+

# 3. we need to select the one running under user gdm (more correct solution)
$ grep -F "gvfs-udisks2-volume-monitor+" /tmp/ps.log | grep -E -m 1 "^gdm[ \t]+"
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+

# 4. naive implementation of the grep command 2. would be
$ awk 'index($0,"gvfs-udisks2-volume-monitor+") > 0 {print $0;exit(0);}' /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+

# 5. more correct implementation of the grep command 3. is
$ awk '$1 == "gdm" && index($0,"gvfs-udisks2-volume-monitor+") > 0 {print $0;exit(0);}' /tmp/ps.log
gdm       1629  0.0  0.1 413944  8428 tty1     Sl  11:46   0:00 /qbin/gvfs-udisks2-volume-monitor+
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

```awk
# 0. text data generation
$ ps auxwww > /tmp/ps.log

# 1. essential head
$ head -3 /tmp/ps.log 
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 194320  8872 ?        Ss   11:44   0:02 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    11:44   0:00 [kthreadd]

# 1.awk mimicking the head command
$ awk 'NR<=3 {print}' /tmp/ps.log
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 194320  8872 ?        Ss   11:44   0:02 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    11:44   0:00 [kthreadd]


# 2. basic tail
$ tail -n 2 /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts9    R+   12:43   0:00 ps auxwww

# 2.awk performing the same in tail is much less straightforward
$ awk -v n=2 'BEGIN{arr[-1]=0}
>                  {arr[arr[-1]]=$0;arr[-1]++;}
>               END{for(i=arr[-1]-n;i<arr[-1];i++){print arr[i]}}' /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts9    R+   12:43   0:00 ps auxwww

```
### Lesson learned:
 * variable `NR` contains number of processed record (acting as record/line number)
 * passing an awk variable from command-line is possible via `-v variable-name=variable-value`
 * associative arrays are fully supported
 * `BEGIN{}` rule is evaluated *before* first line is read
 * `END{}` rule is evaluated *after* last line is processed

## Performing math

### Fibonacci sequence

```awk
# fibo.awk

# actions before reading text stream
BEGIN{
  for(i=0; i<(ARGC > 1 ? ARGV[1] : 10); i++)
    print(fibo_recursive(i));
}

# local functions
function fibo_recursive(in_val) {
    return( in_val<=1 ? in_val : fibo_recursive(in_val-1) + fibo_recursive(in_val-2));
}
```
An execution results in Fibonacci sequence:
```
$ awk -f ~/tmp/fibo.awk 10
0
1
1
2
3
5
8
13
21
34
```

### Text stream statistics

```awk
# wordstats.awk

# actions before reading text stream - initiate counters
BEGIN{
    wcnt = bllinecnt = 0;
    wmin = wmax = "";
}

# at every line ruleblock
{
    # count the words
    wcnt += NF;
    # count blank/empty lines
    if (NF == 0)
        bllinecnt++;
    # update maximum and minimum word count variables
    if( (NF>(wmax+0)) || (NR==1))
        wmax = NF;
    if( (NF<(wmin+0)) || (NR==1))
        wmin = NF;
}

# final text stream statistics
END{
    printf("Total %d words on %d lines. (average/min/max %.2f/%s/%s words per line; %d blank lines)\n",
           wcnt, NR, wcnt / NR, wmin, wmax, bllinecnt);
}
```
The executions:
```
$ echo -e "A B C\nA B" | awk -f ~/tmp/wordstats.awk 
Total 5 words on 2 lines. (average/min/max 2.50/2/3 words per line; 0 blank lines)
$ ps auxwww | awk -f ~/tmp/wordstats.awk
Total 3458 words on 270 lines. (average/min/max 12.81/11/26 words per line; 0 blank lines)
$ curl -s 'https://www.gnu.org/licenses/old-licenses/lgpl-2.1.txt' | awk -f ~/tmp/wordstats.awk
Total 4381 words on 502 lines. (average/min/max 8.73/0/15 words per line; 75 blank lines)
```

### Lesson learned:
 * parsing command-line arguments are possible via C-like variables
   * count of arguments `ARGC` and
   * individual arguments `ARGV` array
 * AWK allow functions to be declared either on the beginning or at the end of awk source code
 * `NF` variable contains number of words (awk fields) on line (awk record)
 * ternary operator `<cond> ? <when-true> : <when-false>` is fully available

## Prettyprinting long awk programs

The rule of thumb says you should place AWK program into separate file if is long and unreadable as the oneliner.

```awk
# 0. text data generation
$ ps auxwww > /tmp/ps.log

# 1. mimicking tail -n 2 
# 1.a as not much readable oneliner
$ awk -v n=2 'BEGIN{arr[-1]=0}{arr[arr[-1]]=$0;arr[-1]++;}END{for(i=arr[-1]-n;i<arr[-1];i++){print arr[i]}}' /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts/9    R+   12:43   0:00 ps auxwww

# 1.b as oneliner on separated lines
$ awk -v n=2 'BEGIN{arr[-1]=0}
>                  {arr[arr[-1]]=$0;arr[-1]++;}
>               END{for(i=arr[-1]-n;i<arr[-1];i++){print arr[i]}}' /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts/9    R+   12:43   0:00 ps auxwww

# 1.c oneliner prettyprinted by "GNU AWK profiler"
$ gawk -p -v n=2 'BEGIN{arr[-1]=0}{arr[arr[-1]]=$0;arr[-1]++;}END{for(i=arr[-1]-n;i<arr[-1];i++){print arr[i]}}' /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts/9    R+   12:43   0:00 ps auxwww
$ cat awkprof.out 
        # gawk profile, created Tue Dec 25 14:48:02 2018

        # BEGIN block(s)

        BEGIN {
     1          arr[-1] = 0
        }

        # Rule(s)

   261  {
   261          arr[arr[-1]] = $0
   261          arr[-1]++
        }

        # END block(s)

        END {
     2          for (i = arr[-1] - n; i < arr[-1]; i++) {
     2                  print arr[i]
                }
        }
# 1.d program in separated file
$ awk -v n=2 -f tail.awk /tmp/ps.log 
f_ii     10347  0.0  0.2 470984 21808 ?        S    12:42   0:00 file.so [kdeinit5] file local:...
f_ii     10354  0.0  0.0 154704  3840 pts/9    R+   12:43   0:00 ps auxwww
```
### Lesson learned:
 * AWK program should be inserted in separate `*.awk` file to avoid poor readability
 * prettyprinting of badly formatted program is posible via GNU AWK "profiling" feature `gawk -p`


## Troubleshooting awk programs

To troubleshoot not functional AWK program you should have GNU AWK in "profiling" mode (`gawk -p`), generated execution summary helps to understand what is wrong.

### Lesson learned:
 * use GNU AWK "profiling" feature `gawk -p` to uncover issues with your awk code
 * if you do not have GNU AWK you need to add custom debugging messages


