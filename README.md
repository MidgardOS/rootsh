# Introduction

`rootsh` is a wrapper for a shell which will make a copy of everything printed
on the terminal.

Its main purpose is to give ordinary users a shell with root privileges while
keeping an eye on what they type. This is accomplished by allowing them to
execute `rootsh` via the `sudo` command or calling rootsh in your shell's
`/etc/profile.d/` scripts.

Unlike a simple `sudo -s` which is the usual way doing this, `sudo rootsh` will
send their terminal keystrokes and output to a logfile and eventually to a
remote syslog server, where they are out of reach and safe from manipulation.

## Motivation

It can vary:

- Sometimes users need to perform tasks on a system which are too complex to be
expressed in sudo rules
- Sometimes there is management pressure to give a user root shell access
- Sometimes you're just tired arguing with users who insist in having root privileges.

With `rootsh` you can give your users access to a root shell while auditing
their actions. 

## Usage:

`rootsh` will be mainly used to give normal users the privilege of a shell
running under uid `0`. This can be accomplished in a couple ways:

1. calling `rootsh` via sudo
1. calling `rootsh` in your shell's start scripts, e.g. in a script under `/etc/profile.d`

### Via `sudo`

For example you have to grant user usr1234 local root privileges on their
workstation ws0001, you can make an entry in a file under `/etc/sudoers.d/`
like this:

```
usr1234		ws0001		= /bin/rootsh
```

They will then have to type the following to become root:

```bash
usr1234@ws0001:~> sudo rootsh
Password:
ws0001:~ # id
uid=0(root) gid=0(root) groups=0(root)
ws0001:~ # 
ws0001:~ # exit
exit
usr1234@ws0001:~>
```

If compiled `rootsh` with the default settings, the keystrokes and output will
be sent line by line to the syslog daemon using priority `local5.info`. To
collect the output coming from running rootsh commands in a specific file make
an entry in your `/etc/syslog.conf` like this:

```
local5.notice	/var/log/rootshell
```

or maybe like this:

```
local5.notice	@your_central_syslog_host
```

The generated syslog entries will look like this:

```
Jul  2 17:44:19 ws0001 rootsh-020a: usr1234=root,/dev/pts/0: logging new rootsh session (rootsh-020a) to /var/log/rootsh/usr1234.20040702174419.020a
Jul  2 17:44:21 ws0001 rootsh-020a: 001: ws0001:~ # id 
Jul  2 17:44:21 ws0001 rootsh-020a: 002: uid=0(root) gid=0(root) groups=0(root)
Jul  2 17:44:22 ws0001 rootsh-020a: 003: ws0001:~ #  
Jul  2 17:46:03 ws0001 rootsh-020a: 004: ws0001:~ # exit 
Jul  2 17:46:03 ws0001 rootsh-020a: 005: exit 
Jul  2 17:46:03 ws0001 rootsh-020a: 006: *** rootsh session ended by user
Jul  2 17:46:03 ws0001 rootsh-020a: usr1234,/dev/pts/0: closing rootsh session (rootsh-020a) 
```

In the above snippet, the `rootsh-020a` is an identifier created from the
program's name and a 4 digit hex number which is the pid of the `rootsh`
process. It will prepend every line sent to syslog and will help you to find
all the entries in a logfile belonging to a specific session.

To find the log information for a session in syslog, follow these steps:

1. find the "logging new..." line for the session you're interested in
1. Take the identifier like `rootsh-020a` in the example and grep all
   occurences of it from your logfile.
1. If `rootsh` is running on many machines, there may be collisions if two
   `rootsh` processes have the same pid. Add the hostname to grep's pattern in
   this case. You will also find the same output locally on the ws0001 host in
   a file called like this `<caller's username>.<timestamp>.<process id>`

Depending on your operating system and configuration parameter `--with-logdir=`
these files can be found in `/var/log/rootsh`, `/var/adm/rootsh` or your own
choice.

The counter after the session identifier can help you find holes if you are not
sure weather logging was incomplete (either due to manipulation or network
problems).

Finished session's logfiles get ".closed" appended to their names. This helps 
with cleaning and archiving the logdir.

The `rootsh` code has some rudimentary tamper detection. If it is triggered,
`rootsh` will attempt tp recreate the file and append ".tampered" instead of 
".closed".

### Executed via `/etc/profile.d` Scripts

Additionally, `rootsh` tool can also be started via profile scripts, like so:

```bash
if [[ -x /usr/bin/rootsh ]]; then
	ppid="$(ps -ho ppid $$)"
	curr_shell="$(ps -ho cmd $$)"
	pshell="$(ps -ho cmd $ppid)"
	if ! echo "${curr_shell} ${pshell}" | grep --quiet 'rootsh'; then
		if [[ "${UID}" == 0 || "${EUID}" == 0 ]]; then
			cmd="/usr/bin/rootsh --no-logfile"
			if [[ "$BASH_EXECUTION_STRING" ]]; then
				cmd="$cmd -- $BASH_EXECUTION_STRING"
			fi
			exec $cmd
		fi
	fi
fi
```

## Additional features

There are a couple other features that `rootsh` has that can be useful,
depending on a site's needs.

### Login Shell Support

`rootsh` can be started as a login shell, using the "-i" parameter.

### Non-root Shell Logging

To run `rootsh` as a different user, the "-u" parameter can be used. When this
option is used the logfile will show "usr1234=root" at the start of the logging.
Without using this option the line will show "root=root" as no other user
information is available.

## How it Works

`rootsh` works very much like the GNU `script` utility. It forks and creates a
parent/child pseudo-terminal (PTY) pair.

The child PTY will become the controlling terminal of the child process which
will execute a shell command. The parent process waits for input from the
user's terminal and sends it to screen via the parent PTY.

All output printed, including the echoed input will be written to a logfile and
to the syslog daemon. **NOTE: this can mean that inadvertently echoed passwords
or other sensitive information may be sent over the wire to the syslog servers!
To avoid bad actors collecting that information over the wire, if being sent to
a remote syslog collector, always encrypt the traffic from the host to the
remote syslog collector system.**

## Warning

THIS APPLICATION COMES WITH NO GUARANTEE OF FITNESS OF PURPOSE! While the code
is pretty straight-forward, there may be ways to escape the auditing mechanizms
provided by `rootsh`. This is why it is highly recommended to use the default
settings for syslog logging, and to use a remote system log storage system that
receives logs over an encrypted communications mechanism.
