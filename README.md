The Early OOM Daemon
====================

[![Build Status](https://api.travis-ci.org/rfjakob/earlyoom.svg)](https://travis-ci.org/rfjakob/earlyoom)

The oom-killer generally has a bad reputation among Linux users. This may be
part of the reason Linux invokes it only when it has absolutely no other choice.
It will swap out the desktop environment, drop the whole page cache and empty
every buffer before it will ultimately kill a process. At least that's what I
think what it will do. I have yet to be patient enough to wait for it, sitting
in front of an unresponsive system.

This made me and other people wonder if the oom-killer could be configured to
step in earlier: [reddit r/linux][5], [superuser.com][2], [unix.stackexchange.com][3].

As it turns out, no, it can't. At least using the in-kernel oom-killer.
In the user space, however, we can do whatever we want.

What does it do
---------------
earlyoom checks the amount of available memory and free swap to to 10
times a second (less often if there is a lot of free memory).
If both are below 10%, it will kill the largest process (highest `oom_score`).
The percentage value is configurable via command line
arguments.

In the `free -m` output below, the available memory is 2170 MiB and
the free swap is 231 MiB.

                  total        used        free      shared  buff/cache   available
    Mem:           7842        4523         137         841        3182        2170
    Swap:          1023         792         231

Why is "available" memory checked as opposed to "free" memory?
On a healthy Linux system, "free" memory is supposed to be close to zero,
because Linux uses all available physical memory to cache disk access.
These caches can be dropped any time the memory is needed for something
else.

The "available" memory accounts for that. It sums up all memory that
is unused or can be freed immediately.

Note that you need a recent version of
`free` and Linux kernel 3.14+ to see the "available" column. If you have
a recent kernel, but an old version of `free`, you can get the value
from `cat /proc/meminfo | grep MemAvailable`.

When both your available memory and free swap drop below 10% of the total,
it will `kill -9` the process that uses the most memory in the opinion of
the kernel (`/proc/*/oom_score`). It can optionally (`-i` option) ignore
any positive adjustments set in `/proc/*/oom_score_adj` to protect innocent
victims (see below).

#### See also
* [nohang](https://github.com/hakavlad/nohang), a similar project like earlyoom,
  written in Python and with additional features and configuration options.
* facebooks's pressure stall information (psi) [kernel patches](https://github.com/facebookincubator/oomd)
  and the accompanying [oomd](https://github.com/facebookincubator/oomd) userspace helper.
  The patches are not not yet merged into the mainline kernel as of 2018-07-21.

Why not trigger the kernel oom killer?
--------------------------------------
Earlyoom does not use `echo f > /proc/sysrq-trigger` because [the Chrome people made
their browser always be the first (innocent!) victim by setting `oom_score_adj`
very high]( https://code.google.com/p/chromium/issues/detail?id=333617).
Instead, earlyoom finds out itself by reading through `/proc/*/status`
(actually `/proc/*/statm`, which contains the same information but is easier to
parse programmatically).

Additionally, in recent kernels (tested on 4.0.5), triggering the kernel
oom killer manually may not work at all.
That is, it may only free some graphics
memory (that will be allocated immediately again) and not actually kill
any process. [Here](https://gist.github.com/rfjakob/346b7dc611fc3cdf4011) you
can see how this looks like on my machine (Intel integrated graphics).

How much memory does earlyoom use?
----------------------------------
About `2 MiB` (`VmRSS`), though only `220 kiB` is private memory (`RssAnon`).
The rest is the libc library (`RssFile`) that is shared with other processes.
All memory is locked using `mlockall()` to make sure earlyoom does not slow down in low memory situations.

Download and compile
--------------------
Easy:

```bash
git clone https://github.com/rfjakob/earlyoom.git
cd earlyoom
make
sudo make install # Optional, if you want earlyoom to start
                  # automatically as a service (works on Fedora)
```

For Arch Linux, there's an [AUR package](https://aur.archlinux.org/packages/earlyoom/):
```bash
yaourt -S earlyoom
sudo systemctl enable earlyoom
sudo systemctl start earlyoom
```

For Debian, there's an [Debian package](https://packages.debian.org/search?keywords=earlyoom):
```bash
apt install earlyoom
```

Use
---
Just start the executable you have just compiled:

```bash
./earlyoom
```

It will inform you how much memory and swap you have, what the minimum
is, how much memory is available and how much swap is free.

```
earlyoom v0.10
mem total: 7842 MiB, min: 784 MiB (10 %)
swap total: 1023 MiB, min: 102 MiB (10 %)
mem avail: 5115 MiB (65 %), swap free: 1023 MiB (100 %)
mem avail: 5115 MiB (65 %), swap free: 1023 MiB (100 %)
mem avail: 5115 MiB (65 %), swap free: 1023 MiB (100 %)
[...]
```

If the values drop below the minimum, processes are killed until it
is above the minimum again. Every action is logged to stderr. If you are on
running earlyoom as a systemd service, you can view the last 10 lines
using

```bash
systemctl status earlyoom
```

### Notifications

The command-line flag `-n` enables notifications via `notify-send`. However, if earlyoom is being
run by a user other than the one running your desktop environment (e.g. if it's run as a service
or cron job) then `notify-send` will not work on its own, as DBUS, X, and/or display information
may required.

In this case, you can use `-N` to supply environment variables or another command. The exact value
will vary depending on your desktop environment, but the following command may work. `YOUR_USER`
should be replaced with output of `whoami` and `YOUR_USER_ID` with output of `echo $UID`. Your
`DISPLAY` value may also be different (check `echo $DISPLAY`).

```bash
earlyoom -N 'sudo -u YOUR_USER DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/YOUR_USER_ID/bus notify-send'
```

Other options are discussed in [this thread](https://unix.stackexchange.com/questions/111188/using-notify-send-with-cron).

Note that if you choose to use a command other than `notify-send`, it must support the same
arguments. Example invocation earlyoom uses:

```bash
NOTIFY_COMMAND -i dialog-warning 'notif title' 'notif text'
```

### Preferred Processes

The command-line flag `--prefer` specifies processes to prefer killing;
likewise, `--avoid` specifies
processes to avoid killing. The list of processes is specified by a regex expression.
For instance, to avoid having `foo` and `bar` be killed:

```bash
earlyoom --avoid '^(foo|bar)$'
```

The regex is matched against the basename of the process as shown
in `/proc/PID/stat`.

Command line options
--------------------
```
./earlyoom -h
earlyoom v1.0-30-gaf48355
Usage: earlyoom [OPTION]...

  -m PERCENT       set available memory minimum to PERCENT of total (default 10 %)
  -s PERCENT       set free swap minimum to PERCENT of total (default 10 %)
  -M SIZE          set available memory minimum to SIZE KiB
  -S SIZE          set free swap minimum to SIZE KiB
  -k               use kernel oom killer instead of own user-space implementation
  -i               user-space oom killer should ignore positive oom_score_adj values
  -n               enable notifications using "notify-send"
  -N COMMAND       enable notifications using COMMAND
  -d               enable debugging messages
  -v               print version information and exit
  -r INTERVAL      memory report interval in seconds (default 1), set to 0 to
                   disable completely
  -p               set niceness of earlyoom to -20 and oom_score_adj to -1000
  --prefer REGEX   prefer killing processes matching REGEX
  --avoid REGEX    avoid killing processes matching REGEX
  -h, --help       this help text
```

See the [man page](MANPAGE.md) for details.

Contribute
----------
Bug reports and pull requests are welcome via github. In particular, I am glad to
accept

* Use case reports and feedback

Changelog
---------
* v1.2, in progress
  * Implement adaptive sleep time (= adaptive poll rate) to lower CPU
    usage further ([issue #61](https://github.com/rfjakob/earlyoom/issues/61))
  * Remove option to use kernel oom-killer (`-k`)
    ([issue #80](https://github.com/rfjakob/earlyoom/issues/80))
  * Gracefully handle the case of swap being added or removed after earlyoom was started
    ([issue 62](https://github.com/rfjakob/earlyoom/issues/62),
    [commit](https://github.com/rfjakob/earlyoom/commit/88e58903fec70b105aebba39cd584add5e1d1532))
  * Implement staged kill: first sigterm, then sigkill, with configurable limits
    ([issue #67](https://github.com/rfjakob/earlyoom/issues/67))
* v1.1, 2018-07-07
  * Fix possible shell code injection through GUI notifications
    ([commit](https://github.com/rfjakob/earlyoom/commit/ab79aa3895077676f50120f15e2bb22915446db9))
  * On failure to kill any process, only sleep 1 second instead of 10
    ([issue #74](https://github.com/rfjakob/earlyoom/issues/74))
  * Send the GUI notification *after* killing, not before
    ([issue #73](https://github.com/rfjakob/earlyoom/issues/73))
  * Accept `--help` in addition to `-h`
  * Fix wrong process name displayed in kill notification
    ([commit](https://github.com/rfjakob/earlyoom/commit/1466e9c8f7997108758d9585442a96c6806c040e))
  * Fix possible division by zero with `-S`
    ([commit](https://github.com/rfjakob/earlyoom/commit/a0c4b26dfef8b38ef81c7b0b907442f344a3e115))
* v1.0, 2018-01-28
  * Add `--prefer` and `--avoid` options (@TomJohnZ)
  * Add support for GUI notifications, add options `-n` and `-N`
* v0.12: Add `-M` and `-S` options (@nailgun); add man page, parameterize Makefile (@yangfl)
* v0.11: Fix undefined behavoir in get_entry_fatal (missing return, [commit](https://github.com/rfjakob/earlyoom/commit/9251d25618946723eb8a829404ebf1a65d99dbb0))
* v0.10: Allow to override Makefile's VERSION variable to make packaging easier,
  add `-v` command-line option
* v0.9: If oom_score of all processes is 0, use VmRss to find a victim
* v0.8: Use a guesstimate if the kernel does not provide MemAvailable
* v0.7: Select victim by oom_score instead of VmRSS, add options `-i` and `-d`
* v0.6: Add command-line options `-m`, `-s`, `-k`
* v0.5: Add swap support
* v0.4: Add SysV init script (thanks [@joeytwiddle](https://github.com/joeytwiddle)), use the new `MemAvailable` from `/proc/meminfo`
  (needs Linux 3.14+, [commit][4])
* v0.2: Add systemd unit file
* v0.1: Initial release

[1]: http://www.freelists.org/post/procps/library-properly-handle-memory-used-by-tmpfs
[2]: http://superuser.com/questions/406101/is-it-possible-to-make-the-oom-killer-intervent-earlier
[3]: http://unix.stackexchange.com/questions/38507/is-it-possible-to-trigger-oom-killer-on-forced-swapping
[4]: https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=34e431b0ae398fc54ea69ff85ec700722c9da773
[5]: https://www.reddit.com/r/linux/comments/56r4xj/why_are_low_memory_conditions_handled_so_badly/
