# Linux

> Notes on the Linux command line, from *The Linux Command Line* (William Shotts).

## Contents

- [Getting Your Bearings](#getting-your-bearings)
- [Navigating the Filesystem](#navigating-the-filesystem)
- [Options and Arguments](#options-and-arguments)
- [Reading Files](#reading-files)
- [System Directories](#system-directories)
- [Manipulating Files and Directories](#manipulating-files-and-directories)
- [Wildcards](#wildcards)
- [Getting Help](#getting-help)
- [What Are Commands?](#what-are-commands)
- [Working with Files](#working-with-files)
- [Shell Features](#shell-features)
- [Processes](#processes)
  - [Everything Is a File Descriptor](#everything-is-a-file-descriptor)
  - [A File Descriptor Can Be a Socket](#a-file-descriptor-can-be-a-socket)
  - [A Socket Has an Address](#a-socket-has-an-address)
  - [LISTEN: The Kernel Holds It for the Process](#listen-the-kernel-holds-it-for-the-process)
  - [One Socket, One Owner](#one-socket-one-owner)
  - [A Process Is a Directory in /proc](#a-process-is-a-directory-in-proc)
  - [Signals Are the Only Thing You Can Do to a Process](#signals-are-the-only-thing-you-can-do-to-a-process)
  - [Escalation: TERM, Wait, Then KILL](#escalation-term-wait-then-kill)
  - [Ctrl+C, Ctrl+Z, and Terminal Hangup](#ctrlc-ctrlz-and-terminal-hangup)
  - [Stopped, Unreaped, and Stuck in I/O](#stopped-unreaped-and-stuck-in-io)
  - [Ceilings: ulimit and nice](#ceilings-ulimit-and-nice)
  - [Docker Adds Namespaces](#docker-adds-namespaces)
  - [What to Run When](#what-to-run-when)

---

## Getting Your Bearings

| Command | Purpose |
|---|---|
| `pwd` | Print the present working directory |
| `cd` | Change the working directory |
| `ls` | List directory contents |

## Navigating the Filesystem

- The **working directory** is assumed whenever a pathname is not given.
- `.` — the working directory. `..` — its parent. So `cd resources` == `cd ./resources`.
- `cd` — go home. `cd -` — go back to the previous directory.

### Filenames

1. Names starting with `.` are **hidden**; `ls` needs `-a` to show them.
2. Names are **case sensitive** (`README` != `readme`).
3. Don't embed spaces in filenames; use an underscore instead.

### ls

| Usage | Meaning |
|---|---|
| `ls dir1 dir2` | List multiple directories (e.g. `ls ~ /something`) |
| `ls -a` | Include hidden files |
| `ls -l` | Long format |
| `ls -t` | Sort by modification time, newest first |
| `ls -lt` | Combine `-l` and `-t` |

```text
-rw-r--r-- 1 rjlinux rjlinux  613 Oct  4 05:09 readme.md
drwxr-xr-x 3 rjlinux rjlinux 4096 Oct  7 11:16 Notes
```

| Field | Meaning |
|---|---|
| `drwxr-xr-x` | Permissions (`-` = ordinary file, `d` = directory) |
| `1` | Hard link count |
| `rjlinux` / `rjlinux` | Owner / group |
| `613` | Size in bytes |
| `Oct 4 05:09` | Last modification |
| `readme.md` | Name |

## Options and Arguments

```text
command -options arguments
```

`command --help` documents any command's options. Example: `ls -lt` is `-l` and `-t` combined.

## Reading Files

`less` opens a file one screenful at a time.

| Key | Action |
|---|---|
| `q` | Quit (also closes `git log` pages) |
| `G` / `g` | Jump to end / beginning of file |

`file filename` reports what a file actually is, regardless of its name.

## System Directories

| Path | Contents |
|---|---|
| `/` | Root — everything starts here |
| `/bin`, `/usr/bin` | Binaries needed to boot and run |
| `/boot` | Kernel, init RAM disk, boot loader |
| `/dev` | Device nodes — every device the kernel knows about appears here as a file |
| `/etc` | System-wide configuration |
| `/home` | Where normal users keep their files |
| `/lib`, `/usr/lib` | Shared libraries used by core programs |
| `/sbin` | System binaries |
| `/tmp` | Temporary files (`/run` is used these days) |
| `/usr/local` | Locally installed software |
| `/usr/share` | Shared data, docs, templates |
| `/var` | Variable data — logs, databases, mail, caches |
| `/sys` | Kernel device and driver state, exposed as files |
| `~/.config`, `~/.local` | Per-user app configuration and program state |

Inside `/etc`:

| File | Purpose |
|---|---|
| `/etc/crontab` | Schedules automated jobs run by `cron` |
| `/etc/fstab` | Storage devices and their mount points |
| `/etc/passwd` | List of user accounts |

Inside `/boot`: `/boot/vmlinuz-*` is the kernel; `/boot/grub/grub.cfg` (or `menu.lst`) configures the boot loader.

## Manipulating Files and Directories

| Command | Purpose |
|---|---|
| `cp` | Copy files and directories |
| `mv` | Move or rename files and directories |
| `mkdir` | Create directories |
| `touch` | Create a file, or update its timestamp |
| `rm` | Remove files and directories |

```bash
cp -u *.html destination_directory   # copy only files missing from, or newer than, those in destination
```

### Options that matter

| Option | `cp` | `mv` | `rm` | Effect |
|---|:-:|:-:|:-:|---|
| `-r` | yes | yes | yes | Recurse into subdirectories — **required** for directories |
| `-i` | yes | yes | yes | Prompt before overwriting or deleting |
| `-f` | yes | yes | yes | Force; no prompts, ignore nonexistent files |
| `-n` | yes | yes | — | Never overwrite an existing file |
| `-v` | yes | yes | yes | Show each file as it is acted on |
| `-u` | yes | — | — | Only copy if newer than the destination copy |

`cp` without `-r` on a directory fails outright. This is the common first stumble:

```bash
cp -r src dst        # copy the directory tree
mv -r src dst        # move the directory tree
cp -i big.iso backup/   # ask before clobbering
rm -r old_logs           # delete a directory tree
```

`rm` bypasses the trash and there is no undo. Two habits worth keeping:

```bash
rm -i file        # or -n, or -v — pick one
ls file && rm file   # confirm the wildcard expanded to what you expected
```

`rm` will not remove a directory without `-r`, and `-f` combined with `-r` will silently succeed even
if nothing matched.

## Wildcards

Patterns the shell expands into filenames before the command runs.

| Wildcard | Matches |
|---|---|
| `*` | Any characters, including none |
| `?` | Any single character |
| `[characters]` | Any character in the set |
| `[!characters]` / `[^characters]` | Any character **not** in the set |
| `[[:class:]]` | Any character in the named class |

Classes: `[:alnum:]` alphanumeric, `[:alpha:]` alphabetic, `[:digit:]` numeral, `[:lower:]` lowercase, `[:upper:]` uppercase.

| Pattern | Matches |
|---|---|
| `*` | All files |
| `g*` | Any file beginning with `g` |
| `b*.txt` | Any `b...` file ending in `.txt` |
| `Data???` | `Data` plus exactly three characters |
| `[abc]*` | Any file beginning with `a`, `b`, or `c` |
| `BACKUP.[0-9][0-9][0-9]` | `BACKUP.` plus exactly three numerals |
| `[[:upper:]]*` | Any file beginning with an uppercase letter |
| `[![:digit:]]*` | Any file not beginning with a numeral |
| `*[[:lower:]123]` | Any file ending in a lowercase letter or `1`, `2`, `3` |

## Getting Help

| Command | Purpose |
|---|---|
| `man` | Display a command's manual page |
| `help` | Get help for shell builtins |
| `apropos` | Display a list of appropriate commands |
| `info` | Display a command's info entry |
| `whatis` | Display one-line manual page descriptions |
| `which` | Display which executable program will be executed |
| `type` | Indicate how a command name is interpreted |

## What Are Commands?

A "command" is any of four things:

1. An **executable program**
2. A **shell builtin**, e.g. `cd`, `mkdir`
3. A **shell function** or script
4. An **alias** — created with `alias`

`type` tells you which of the four you're actually running.

## Working with Files

| Command | Purpose |
|---|---|
| `cat` | Concatenate files |
| `sort` | Sort lines of text |
| `uniq` | Report or omit repeated lines |
| `grep` | Print lines matching a pattern |
| `wc` | Print newline, word, and byte counts |
| `head` | Output the first part of a file |
| `tail` | Output the last part of a file |
| `tee` | Read from stdin, write to stdout *and* files |

### Options that matter

`grep`:

| Option | Effect |
|---|---|
| `-i` | Ignore case |
| `-v` | Invert — print lines that **do not** match |
| `-c` | Print only a count of matching lines |
| `-n` | Prefix each line with its line number |
| `-r` | Recurse into subdirectories |
| `-l` | Print filenames only, no matching lines |
| `-E` | Extended regex (`\|`, `+`, `?` without escaping) |

`sort`:

| Option | Effect |
|---|---|
| `-n` | Numeric order, not lexicographic — without it `10` sorts before `9` |
| `-r` | Reverse |
| `-u` | Unique lines, dropping duplicates |
| `-h` | Human-readable sizes |
| `-k` | Sort by a specific field or column |

`uniq`: only collapses **adjacent** duplicates, which is why it always follows `sort`.

`wc`: `-l` lines, `-w` words, `-c` bytes, `-m` characters.

### Pipelines

A pipe sends one command's stdout into the next command's stdin. Each command does one small thing;
the composition is the power.

```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -n   # histogram of login shells
grep -c 'ERROR' app.log                                    # how many errors
grep -v '^#' config.yaml                                  # config without comments
sort -nr data.txt | head -10                               # top 10 lines numerically
find . -name '*.log' | xargs grep -l 'Exception'          # which logs contain an error
```

`sort -n` before `head` is the pattern to remember: raw `sort` would rank `9` above `10`.

`tee` is the odd one out — it copies stdin to both stdout and files, so you can watch output *and*
capture it at once:

```bash
make 2>&1 | tee build.log
```

`wc` on a pipeline counts what flowed through it, which makes it a quick sanity check:

```bash
cat *.log | wc -l      # total lines across all logs
```

## Shell Features

**Arithmetic expansion**

```bash
echo $((2+3))
5
```

**Brace expansion** — happens before the command runs

```bash
echo Front-{A,B,C}-Back
Front-A-Back Front-B-Back Front-C-Back
```

**Parameter expansion** — the shell stops the variable name at the first non-alphabetic character, so
`$1hello` is parameter `1` followed by literal `hello`. Use `${var}` when the name runs into adjacent
text.

```bash
echo $1hello
hello
```


---

## Processes

Most Linux notes are a list of commands. This section is the opposite: one chain of causation, with
each command appearing only where a concept requires it.

Start with the case everyone hits. A server won't start and says:

```bash
$ python3 app.py
OSError: [Errno 98] Address already in use
```

You reach for `sudo lsof -i :8080` and get a PID. That answer is useful but thin — it tells you
*what* to kill, not why a number in a config file has anything to do with a running program. The chain
below is the whole answer:

```text
8080                          a number in a config file
  ↓
TCP port                     a 16-bit number naming one end of a connection
  ↓
listening socket              the kernel object bound to that number
  ↓
process owns the socket       the socket is a file descriptor in some process
  ↓
PID identifies the process    an integer, and a directory in /proc
  ↓
process can receive signals   the only way to ask it to do anything
```

Read the rest of this section as the unpacking of that chain, top to bottom.

### Everything Is a File Descriptor

The bottom of the stack, because everything above it depends on it. A running process has a table of
integers. Each integer is a slot pointing at something the kernel calls an **open file** — a regular
file, a pipe, a device, a socket. The slot number is a **file descriptor**, or fd.

Three descriptors exist in every process, and you already know them by another name:

| fd | Name | Default target |
|---|---|---|
| 0 | stdin | the terminal |
| 1 | stdout | the terminal |
| 2 | stderr | the terminal |

This is why redirection works at all. `echo hi > file` doesn't ask `echo` to know about files; the
shell redirects **fd 1** to the file, and every program that writes to stdout goes there. A program
never needs to be told where its output goes.

The descriptor table is a directory:

```bash
$ ls -l /proc/self/fd
lr-x------ 1 rjlinux rjlinux 64 Oct  7 12:56 0 -> /dev/null
l-wx------ 1 rjlinux rjlinux 64 Oct  7 12:56 1 -> pipe:[2624208]
l-wx------ 1 rjlinux rjlinux 64 Oct  7 12:56 2 -> pipe:[2624208]
```

`/proc/self` means "this process" — a self-reference that works for any process, so you never need to
know your own PID to look yourself up.

Read those three lines and you've read the output of the pipeline you ran to produce them. stdout and
stderr both point at `pipe:[2624208]` — the same anonymous pipe. That pipe is the *other end* of a
socket, or the receiving side of a `|`. Two processes share it. There is no kernel object called "a
pipeline"; there are two processes and a pipe between their fd 1s.

One consequence worth holding onto: **closing a terminal closes fds 0, 1, and 2.** Remember that for
the hangup discussion at the end.

### A File Descriptor Can Be a Socket

A socket is one of the kinds of thing a descriptor can point at. But it's a different *kind* of thing
from a file: a file has bytes at offsets, you can seek it, reading it twice gives you the same bytes. A
socket is a **conversation with the network** — it has a peer, and bytes arrive in order from that peer
and nowhere else.

That asymmetry is the whole reason sockets need an address. A file is found by a path the kernel resolves
to a location on disk. A socket's peer is somewhere else entirely — possibly on another machine — so it
needs a way to be named.

Descriptors are how a program talks to the outside world, and this is why a program can hold several
sockets open at once: fd 3, fd 4, fd 5, each one a different connection. When you see a server process
with 500 open descriptors, it has 500 things it can be talking to. `lsof` exists to name them.

### A Socket Has an Address

A socket address is a pair: an IP address and a port number. Both halves are necessary. The IP says
*which machine*; the port says *which conversation* on that machine. Two servers can bind the same
port on different IPs without conflict, and can bind different ports on the same IP.

A **port** is 16 bits, so 0–65535. The ranges are assigned:

| Range | Who |
|---|---|
| 0–1023 | Standard/system. Binding needs root |
| 1024–49151 | Registered to well-known services |
| 49152–65535 | **Dynamic/ephemeral** |

The ephemeral range explains something you see constantly in `ss` output. When *you* curl a server,
your machine picks a port from that range for the connection. So the server's log shows the client's
port changing on every request, while the client sees the server's port stay fixed. The server is the
one whose port is stable, and it's usually the low one.

Port 80 needs root; 8080 doesn't. That's the entire reason software ships on 8080 and 443 instead —
not an aesthetic choice, a permissions one.

### LISTEN: The Kernel Holds It for the Process

```bash
$ ss -tlnp
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      1000   10.255.255.254:53         0.0.0.0:*
LISTEN 0      511    127.0.0.1:35737      0.0.0.0:*    users:(("MainThread",pid=1357,fd=30))
LISTEN 0      4096       0.0.0.0:27017      0.0.0.0:*
LISTEN 0      4096          0.0.0.0:8080       0.0.0.0:*
```

Each row is a socket. `State` is what the socket is doing, and it's the field that answers "is this the
problem?" A `LISTEN` socket has accepted no connections — it's an open door. An `ESTABLISHED` socket is
a live conversation.

`LISTEN` means the owning process has asked the kernel to hold this address open and queue incoming
connections. The *socket* exists because the process opened it; the port is reserved because the kernel
promised to.

Read the **Local Address** column next, because it decides reachability and causes more confusion than
anything else in this section:

| Binding | Who can reach it |
|---|---|
| `127.0.0.1:8080` | Only this machine |
| `0.0.0.0:8080` | Everything |
| `10.255.255.254:53` | One specific interface |

`0.0.0.0` is not an address — it's the absence of one, meaning "every interface." A server bound to
`127.0.0.1` is answering the door but the door faces the wall. "Works on my machine, not from my phone /
not in Docker / not over the network" is nearly always this.

`Recv-Q`/`Send-Q` are bytes queued in the kernel. On a `LISTEN` socket, a growing `Recv-Q` means clients
are arriving faster than the process accepts them.

The other connection states exist and two are worth knowing:

| State | What it means |
|---|---|
| `ESTABLISHED` | Live conversation |
| `TIME_WAIT` | Just closed; held briefly so late packets don't corrupt a new connection on the same port |
| `CLOSE_WAIT` | **The peer is done. Your side hasn't closed.** A leak in your code |

`TIME_WAIT` lasting a minute after a normal disconnect is correct and harmless. `CLOSE_WAIT` is the one
that matters — it means your program isn't closing sockets, and you'll see thousands accumulate.

### One Socket, One Owner

You now have everything needed for `EADDRINUSE`.

The kernel permits **one** socket per (protocol, address, port). Your server asked to bind
`0.0.0.0:8080`; the kernel said no, because a socket already holds that exact combination. Somebody
else got there first.

Confirm it with:

```bash
$ sudo lsof -i :8080
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3  8821 rjlinux 6u  IPv4  ...       0t0  TCP *:8080 (LISTEN)
```

`lsof` walks the fd table of every process on the system and prints the ones that are sockets —
`lsof -i` is shorthand for "sockets only." That PID is the owner. `ss -tlnp | grep :8080` says the same
thing from the other direction, scanning sockets instead of processes. `fuser -k 8080/tcp` kills it in
one step.

So the command answers the question, and the model explains it: **the port is not a number in a file,
it's a socket, and sockets belong to processes.** `Errno 98` is the kernel telling you a process already
holds it.

Two subtleties that explain confusing cases:

- Binding `0.0.0.0:8080` and `127.0.0.1:8080` simultaneously *is* allowed — a wildcard binding doesn't
  conflict with a specific one on the same port. Two specific addresses on one port is not allowed.
- If nothing is listening and you still get `EADDRINUSE`, the old socket is in `TIME_WAIT`. A server
  sets `SO_REUSEADDR` to tolerate this, which is why restarting a dev server usually works and sometimes
  doesn't.

### A Process Is a Directory in /proc

You've gone from a number to a socket to the process holding it. The remaining question is what a
process *is*.

Answer: a **process** is a running program — code in memory, plus a kernel-owned record of its identity,
resources, and open descriptors. The kernel exposes that record as a directory:

```bash
ls /proc/8821
```

| Entry | What's in it |
|---|---|
| `cmdline` | The arguments, NUL-separated |
| `status` | PID, PPID, state, uid, gid, threads, memory |
| `cwd` | A symlink to its working directory |
| `exe` | A symlink to the actual binary |
| `fd/` | Its descriptor table |
| `environ` | Its environment variables |

```bash
$ readlink /proc/self/exe
/usr/bin/python3
$ readlink /proc/self/cwd
/home/rjlinux/Fieldnotes
```

`/proc` is not a real filesystem on disk — it's a live window into the kernel. The entries exist only
while the process does. **That's the real content of the PID:** an integer that indexes a directory,
and a name for that directory in every command that deals with processes.

A PID is unique at any moment and reused after the process exits, so an old PID in a log means nothing
on its own.

`cmdline` is worth one trick, because it settles arguments that `ps` reformats:

```bash
$ tr '\0' ' ' < /proc/self/cmdline
```

`ps` joins a long command line into a single readable column. `/proc/PID/cmdline` shows the arguments
exactly as they were passed.

**Programs vs processes.** Ten browsers open is ten processes of one program, each with its own PID,
memory, and descriptor table. That's why "the process" is ambiguous until you have a PID.

**Where a PID comes from.** When you type a command, the shell forks a copy of itself and that copy
execs the program. So a process's **PPID** (parent PID) is the shell — visible in `ps -ef`, and the
reason scripts can find the shell that launched them.

**Parent death.** If a parent exits, its children keep running and are reparented to PID 1. This is why
a server started over SSH frequently survives the disconnect and holds its port afterwards: nothing told
it to stop.

Threads are processes with a shared address space. In `pstree`, `{caddy}(332)` is a thread — its own PID,
own descriptors, same memory. When you count "processes" and see more rows than you expect, `ps -eLf`
is what separates them.

To list processes, `ps aux` is enough to start; `%CPU` is average since start, `RSS` is RAM actually
used (`VSZ` is address space *reserved* and is routinely meaningless), and `TIME` is **CPU time
consumed, not elapsed**. That last one explains why a server can be up for six hours showing `TIME
0:04` — it was waiting on network, not burning CPU.

`ps aux | grep node` matches its own grep line; `pgrep -a node` doesn't.

### Signals Are the Only Thing You Can Do to a Process

Here is the actual constraint, and it's the one that surprises people: **you cannot reach into a running
process.** Not to change a variable, not to call a function, not to ask what it's thinking. You can't
even ask it a question and get a real answer.

The only channel is a **signal**: an asynchronous message delivered to the process by the kernel. A
signal carries almost no data — not a payload, not a return value. It's a doorbell, and the receiving
program decides what it means.

The process can be notified of exactly these:

- `SIGTERM` (15) — please shut down. The polite request
- `SIGINT` (2) — interrupt. This is `Ctrl+C`
- `SIGQUIT` (3) — abort and dump core (`Ctrl+\`)
- `SIGHUP` (1) — hang up. Also conventionally "reload your config"
- `SIGUSR1` / `SIGUSR2` (10 / 12) — user-defined. `SIGUSR1` commonly means "reopen your log file"
- `SIGSTOP` (19) / `SIGCONT` (18) — stop and resume, like a pause
- `SIGCHLD` (17) — a child of mine exited
- `SIGKILL` (9) — die

Full list: `kill -l`.

Signals and their interactivity:

| Signals | Catchable? |
|---|---|
| Most (`TERM`, `INT`, `HUP`, `USR1`) | Yes — the program can handle, block, or ignore them |
| `SIGKILL`, `SIGSTOP` | **No.** The kernel acts regardless |

That asymmetry is the design. A program can refuse to be asked nicely — which is why `Ctrl+C` sometimes
does nothing — but it cannot refuse to be killed. `SIGKILL` exists so that a wedged process is never
irrecoverable.

So a signal is only a *request*. `SIGTERM` doesn't terminate anything by itself; it hands the program a
decision. What happens next is entirely the program's choice, and that's why signal handling is one of
the more reliable ways to tell a well-built server from a badly built one.

Signals go to a *process*, a *process group*, or a session. `Ctrl+C` goes to the whole foreground
process group — which is why a pipeline stops all at once instead of one stage at a time.

### Escalation: TERM, Wait, Then KILL

Sending the signals:

```bash
kill 8821              # SIGTERM by default
kill -TERM 8821        # explicit, clearer
kill -9 8821           # SIGKILL
kill -INT 8821
kill -s SIGINT 8821

pkill -TERM nginx       # by process name
pkill -f 'worker.*-t'  # match the full command line
killall -u rjlinux node
pkill -u rjlinux -f pytest
```

The pattern to internalize, because defaulting to `-9` breaks things:

```bash
kill -TERM 8821
sleep 5
kill -0 8821 2>/dev/null && kill -9 8821     # still there? escalate
```

`SIGTERM` lets a server finish what it's doing: drain in-flight requests, close database connections,
flush buffers. `SIGKILL` gives it nothing — the process disappears between instructions. For a database
that means a crash-recovery cycle on next start; for a network server, dropped connections.

`kill -0 <pid>` sends **no** signal. It only asks the kernel "does this process exist and may I signal
it?" It's the safest way to test a PID, and it's what makes the escalation loop work.

Killing by name needs care: `pkill python` matches every Python process, including the one you want.
`pkill -f` matches the whole command line and is more precise. Check with `pgrep -a` before you commit.

### Ctrl+C, Ctrl+Z, and Terminal Hangup

The keyboard is wired to signals, which is why job control feels like process control.

| Key | Signal | Effect |
|---|---|---|
| `Ctrl+C` | `SIGINT` | Interrupt the foreground job |
| `Ctrl+Z` | `SIGTSTP` | Suspend it, return to the prompt |
| `Ctrl+\` | `SIGQUIT` | Abort and dump core |

A process in the foreground **owns your terminal**. That's why only one job can run in the foreground at
a time, and why a server that never exits leaves you unable to type anything.

```bash
$ python3 app.py
^C                 # SIGINT — but suppose it's ignored
^Z
[1]+  Stopped                 python3 app.py
$ jobs -l
[1]+  8821 Stopped            python3 app.py
$ kill -TERM %1               # or bg %1 to resume in the background
$ fg %1                       # back to the foreground
$ wait                        # block until children exit
$ disown %1                   # drop it from the shell's job table
```

`%1` refers to the first job, `%1` alone means the most recent. Job control (`jobs`, `fg`, `bg`, `wait`)
only exists in an **interactive shell attached to a real terminal**. In a script or a non-interactive
session these behave differently or aren't there at all — which is why `wait` in a cron job or a Docker
entrypoint does something other than what you'd expect.

**Hangup.** Close the terminal and the kernel sends `SIGHUP` to the foreground process group. Remember
that closing a terminal closes fds 0, 1, 2 — so a process whose stdout was the terminal loses its
output too. `SIGHUP` is the polite form of that, and most programs exit on it. This is why:

- A server started over SSH often survives the disconnect. It got reparented to PID 1, and nothing sent
  it `SIGHUP`.
- A shell script that started children may leave them orphaned rather than cleaned up.

To outlive the terminal, you have to defeat that signal:

```bash
tmux new -s dev          # the right answer; tmux is installed here
# Ctrl+B then D to detach, tmux attach -t dev to return

nohup python3 app.py > app.log 2>&1 &
setsid python3 app.py < /dev/null > app.log 2>&1 &
```

`nohup` ignores `SIGHUP`. `setsid` detaches into a new session with no controlling terminal at all.
The `> app.log 2>&1 &` matters as much as the `nohup` — without it the output goes to `nohup.out` in
whatever directory you happened to be standing in.

Both keep a process alive. **systemd supervises** one:

```bash
systemctl start myapp
systemctl status myapp
systemctl enable myapp        # start at boot
journalctl -u myapp -f        # follow its logs
```

That's a different guarantee: restart on failure, ordering against dependencies, structured logs. Use
`tmux` for a dev server you're about to kill anyway; use systemd for anything that should survive a
reboot.

### Stopped, Unreaped, and Stuck in I/O

The one-letter state in `ps`'s `STAT` column decodes the ways a process can be present but not running.

| Letter | State | Notes |
|---|---|---|
| `R` | Running or runnable | Actively on a CPU |
| `S` | Sleeping | Waiting for I/O. **The normal state** |
| `D` | Uninterruptible I/O | Waiting on hardware. **Signals will not work** |
| `T` | Stopped | By `SIGSTOP`/`Ctrl+Z` |
| `Z` | Zombie | Exited; the parent hasn't collected it yet |

The second letter adds `+` (foreground of a terminal), `l` (multithreaded), `s` (session leader).

**Zombies.** When a process exits, its exit status becomes a small piece of data the parent is expected
to read. Until the parent does, the kernel keeps a record — a zombie. It holds a PID slot and nothing
else: no memory, no CPU. You cannot kill a zombie; it's already dead. The only fix is to make its parent
read the status, or kill the parent so the zombie is reparented to PID 1, whose one job is to collect
adopted children.

```bash
$ ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

An application with hundreds of zombies has a parent that never calls `wait()`.

**`D` state** means the kernel is blocked in the driver and the process cannot be interrupted — not even
by `SIGKILL`. A long-lived `D` means the storage underneath has stopped responding. This is the one case
where "just SIGKILL it" produces nothing at all.

### Ceilings: ulimit and nice

Two ways to constrain a process, both enforced by the kernel at process-creation time.

`ulimit -a` prints the limits the shell hands to anything it starts:

```text
open files    (-n) 1048576
max user processes (-u) 30977
virtual memory (-v) unlimited
cpu time      (-t) unlimited
```

| Limit | Failure when hit |
|---|---|
| `-n` open files | `Too many open files` |
| `-u` processes | `fork: retry: Resource temporarily unavailable` |
| `-v` memory | Process refused to start |
| `-t` cpu time | Killed when it hits the cap |

```bash
ulimit -n              # read
ulimit -n 4096         # raise the soft limit
ulimit -Sn unlimited   # soft only; the hard limit needs root
```

Every socket and open file costs one descriptor, which is why a connection-heavy server hits the
default 1024 long before it hits a memory limit.

`nice` moves a process along the CPU-scheduling priority scale, where **higher means less CPU**:

```bash
nice -n 10 python3 batch.py    # start deprioritized
renice -n 5 -p 8821           # change a running process
```

Lowering the number requires root. Raise it for a batch job that shouldn't fight your interactive work.

### Docker Adds Namespaces

A container is not a virtual machine. It's a set of **namespaces** — separate kernel views — plus
cgroups for limits. The processes inside are ordinary host processes; the namespaces are what make
`ps aux` inside show a different world.

**PID namespace** — the container gets its own PID space starting at 1. So:

```bash
docker exec -it web ps aux     # container-local PIDs: your app is 1
docker top web                 # host view, host PIDs
```

**PID 1 has two special responsibilities**, and skipping them is the source of most weird container
behavior:

1. **It must reap orphans.** When any process in the container dies, PID 1 is expected to collect it. A
   shell or a bare app as PID 1 doesn't, so zombies accumulate — the classic "container gets slower and
   leaks memory" symptom. `docker run --init` injects `tini` to do it.
2. **Signal handling is on the app.** `docker stop` sends `SIGTERM` to PID 1. Many runtimes handle it,
   some don't. Those that don't get `SIGKILL` ten seconds later, and lose the graceful shutdown.

```bash
docker stop web        # SIGTERM to PID 1, wait 10s, then SIGKILL
docker kill web        # SIGKILL immediately
docker kill -s HUP web
docker restart web
```

The ten-second window in `docker stop` is what makes draining in-flight requests possible. `docker kill`
skips it, which is exactly when you see 502s.

**Ports.** A container's network is its own namespace, so a server listening on `0.0.0.0:8080` *inside*
is invisible to the host until published:

```bash
docker run -p 8080:8080 img                  # host 8080 → container 8080
docker run -p 127.0.0.1:8080:8080 img        # host loopback only
docker run -p 8080:80 img                    # host 8080 → container 80
docker port web 8080                         # what's mapped
```

Left number is the host port, right is the container's. Binding the host side to `127.0.0.1` is the
difference between a dev server only your machine can reach and one exposed to the network — and it
should be the default for anything running untrusted code.

Note this composes with the chain at the top: the container's process owns a socket in the container's
network namespace, and `docker-proxy` (or an iptables rule) owns another socket on the host, holding
the host port. Same mechanism, one hop more.

**Limits** are cgroups, the container form of `ulimit`:

```bash
docker run --memory=512m --cpus=1.5 --pids-limit=100 img
docker stats
```

Exceeding `--memory` gets the process OOM-killed: no log output, exit code 137. Confirm with
`docker stats` or the `OOMKilled` field in `docker inspect`.

**Killing something wedged inside:**

```bash
docker exec web kill -TERM 42
docker exec web kill $(docker top -eo pid,cmd | grep worker | awk '{print $1}')
```

### What to Run When

Not a reference table — the commands that fall out of the chain, in the order you'd actually reach for
them.

| You want to know | Run | Why it works |
|---|---|---|
| Who's holding this port? | `sudo lsof -i :8080` | Reads every process's fd table |
| ...same, faster | `ss -tlnp \| grep :8080` | Reads the socket table instead |
| ...kill it in one step | `fuser -k 8080/tcp` | Finds the fd, kills the owner |
| Can I reach this server from elsewhere? | `ss -tlnp` → read Local Address | `127.0.0.1` means only this machine |
| What is this process? | `readlink /proc/PID/exe` | The actual binary, not what `ps` displays |
| What files does it have open? | `ls -l /proc/PID/fd` | Its descriptor table |
| Where is it connected? | `ss -tnp \| grep PID` | Its outbound sockets |
| Is it still running? | `kill -0 PID` | Sends no signal; just tests |
| Stop it properly | `kill -TERM PID`, wait, then `-9` | Give it a chance to clean up |
| Am I leaking sockets? | `ss -tan state close-wait` | `CLOSE_WAIT` means you aren't closing |
| Will it survive my terminal? | `tmux new -s name` | Owns its own terminal |

If you keep a mental model, you don't need most of this. You need it when the model says a place to look,
and then you look there.
