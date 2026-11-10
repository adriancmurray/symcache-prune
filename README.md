# symcache-prune

TLDR: on some Macs, `/System/Library/Caches/com.apple.coresymbolicationd` grows by gigabytes per hour and macOS
shows it only as "System Data". `symcache-prune` is one zsh script that installs an hourly launchd job which empties
that folder once it reaches 10 GB (or a size you choose). Check yours first:

```sh
sudo du -sh /System/Library/Caches/com.apple.coresymbolicationd
```

## The problem

`coresymbolicationd` is the macOS daemon that turns addresses in binaries into symbol names, for crash reports,
debuggers, profilers and test tools. It keeps a cache, mostly one file named `data`, in
`/System/Library/Caches/com.apple.coresymbolicationd`. Storage settings do not name it; it is counted as part of
"System Data". On one example machine, a development Mac running many test runs, the cache went from 0 to 9.4 GB in
about 5 hours and later reached 39 GB. Those are one machine's numbers; yours may grow much more slowly or not at all.

## Check yours

```sh
sudo du -sh /System/Library/Caches/com.apple.coresymbolicationd
ls /Library/Logs/DiagnosticReports/coresymbolicationd_*.diag
```

The folder is readable only by root, so `du` needs `sudo`. It only reads sizes. The same command is printed by
`./symcache-prune check`.

The `.diag` files are resource reports. macOS writes one when a process writes more to disk than its allowed budget
over a period of time. A `coresymbolicationd` report names the event as disk writes, gives how much the daemon wrote
and over how long, the limit it went over, and stack samples of the code doing the writing. If you have several of
these, the cache is probably growing on your machine.

## Clear it by hand

```sh
sudo find -x /System/Library/Caches/com.apple.coresymbolicationd -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
```

This deletes what is inside the folder and keeps the folder itself. The daemon rebuilds the cache as it needs it.

## Install, uninstall, status

Read the script before you run it as root; it is under 200 lines.

```sh
sudo ./symcache-prune install        # threshold 10 GB
sudo ./symcache-prune install 20     # threshold 20 GB; run again to change it
./symcache-prune status              # is the job loaded, last 5 log lines
sudo ./symcache-prune status         # the same, plus the cache size
sudo ./symcache-prune uninstall      # unload the job, remove the plist, the copy and the log
```

`install` writes:

- `/usr/local/libexec/symcache-prune`, a copy of the script (root:wheel, mode 755);
- `/Library/LaunchDaemons/local.symcache-prune.plist` (root:wheel, mode 644), label `local.symcache-prune`;
- `/var/log/symcache-prune.log`, created at the first prune.

`run [threshold-gb]` is what launchd calls. You can also call it yourself with `sudo`.

Run `zsh test` to check the script. It uses a temporary folder, needs no root and changes nothing on the system.

## How it works

Every hour, and once when the job loads, launchd runs `/usr/local/libexec/symcache-prune run` as root. The script
measures the cache with `du -skx`. Below the threshold it does nothing and logs nothing. At or above it, it deletes
everything inside the folder, measures again, and appends one line to the log:

```
<UTC time> before=<size before, MB>MB freed=<MB freed>MB
```

It refuses to act when the cache path is a symlink or not a directory, and exits with code 2. The cache path is
fixed in the script; no argument or environment variable can change it.

## Why it's built this way

- **Measure with du, delete at 10 GB, leave the daemon running.** WebKit's CI machines hit the same growth
  ([WebKit bug 291979](https://bugs.webkit.org/show_bug.cgi?id=291979)). Their fix,
  [delete-if-too-large](https://github.com/WebKit/WebKit/blob/main/Tools/Scripts/delete-if-too-large), measures the
  folder with `du` and deletes it once it reaches 10 GB, without stopping `coresymbolicationd` first. This tool
  copies that approach, including the 10 GB default.
- **Deleting it is safe.** An Apple engineer on the
  [Developer Forums](https://developer.apple.com/forums/thread/652737) described the folder as a system-wide cache of
  symbol information, shared by the tools that look symbols up. Losing it does no harm beyond making the next lookups
  slower while it is rebuilt.
- **A job instead of a setting.** As far as we found there is no supported size cap, environment variable or
  `defaults` key for this cache. If one exists, we did not find it.
- **What fills it.** On the example machine, the reports' "On Behalf Of" lines name the client processes the daemon
  wrote for. `xctest` (test runs on freshly built binaries) appeared in 6 of 7 reports, with `spindump` and `sample`
  in a few; most client pids were short-lived and listed as UNKNOWN, so this is a strong hint, not proof. The
  daemon's own stack shows it serializing symbol archives that clients send it, so every new build plausibly adds
  new entries. That is one machine's evidence.
- **A root-owned copy, with its folders checked.** A root job must never run a file a normal user can replace, or
  that user could run code as root. So `install` copies the script to `/usr/local/libexec` as root:wheel 755 and the
  job runs that copy, never the one you downloaded. Before copying, it checks that `/usr/local/libexec`,
  `/usr/local` and `/usr` are owned by root and not writable by group or others. On some Macs `/usr/local` belongs
  to the user (Intel Homebrew does this); there `install` refuses rather than install a file a user could swap.
- **`#!/bin/zsh -f`.** The `-f` flag stops zsh from reading startup files such as `/etc/zshenv` or `~/.zshenv`, so
  nothing outside the script runs inside the root job.
- **Hourly, at low priority.** The job checks once an hour with `Nice 10` and `LowPriorityIO`, so measuring a large
  cache does not compete with your own work. At a few GB per hour, hourly keeps the cache near the threshold.
- **Delete the contents, not the folder.** The folder is created and owned by the system. Keeping it, with its
  owner and permissions, means the daemon finds things as it expects. `find -x` stays on the cache's volume, and
  `rm` removes symlinks inside it without following them.

## What has NOT been verified

- **macOS versions.** Written and run on one Mac (macOS 27.3 beta, build 26C5035r). Other versions are untested; the cache path
  or its behaviour may differ.
- **SIP.** We have not checked how System Integrity Protection, on or off, affects deleting inside this folder.
  If SIP blocks it, the job fails and writes the error to the log.
- **Deleting while the daemon writes.** The script does not stop `coresymbolicationd`, following WebKit. We have not
  checked what happens to a lookup that is writing to the cache at the moment it is deleted.
- **Whether macOS trims it itself.** We have not seen macOS shrink this cache on its own, but we have not tested it
  under low-disk conditions or over long periods without the job.
- The test suite covers the pruning logic, the plist and the argument checks against a temporary folder. `install`,
  `uninstall` and `status` as root are not covered by it.
