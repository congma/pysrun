# pysrun

pysrun: a custom client for authenticating to BNU's campus Internet system,
written in Python.


## USAGE

### Installation

The script is self-contained and no other packages are necessary, as long as
you have an up-to-date and complete installation of Python.  Two files are
needed:

1. `pysrun`: Python executable,
2. The configuration file, default path at `~/.pysrun.cfg`.

An example configuration file, `pysrun.cfg.default`, is included in the 
package.  It is recommended you copy this file to `~/.pysrun.cfg` and fill in
the login information.  The executable file `pysrun` can be put anywhere but
usually should be copied to a directory included in the `PATH` environment
variable.

The configuration file is not strictly needed, for you can use the equivalent
command-line options.  However, it is convenient to store the options in the
file because you rarely need to change them.


### Configuration file

The configuration file has the following sections, each containing several
options.

1. `[Server]`: Information regarding the authentication server of BNU.  You
don't need to alter this unless you know what you are doing.
 * `address`: Host name or IP address of the server.  Default: 172.16.202.201.
 * `port`: Port number of the service.  Default: 3333.
2. `[Client]`: Client-side information needed by the service.
 * `interface`: Network interface for which the MAC address will be collected
 as part of the required authenticating information.  Default: eth0.  Tip: use
 a command like `ifconfig` to get the name of the interface.
3. `[Account]`: User account information.
 * `username`: Account number as given by your campus ID.  No default value.
 * `password`: Your password.  No default value.
4. `[Session]`: Log-in session information.
 * `uidfile`: Path to the so-called "uid" file.  The "uid" is a session ID
 returned by the server on each log-in.  It is required when you log out.  The
 path indicated by this `uidfile` option is used to store the "uid".  This file
 should not be removed or altered before log-out.  Default: `~/.pysrun_uid`.

The format of the configuration file is defined by the following rules:

1. The file is comprised of lines.  A valid line is of the form `key = value`.
Key, the equal (`=`) sign, and value can be surrounded by an arbitrary
number of whitespace characters such as the space or tab.  Whitespace can be
part of the key or the value provided that it does not start or end one.  If
multiple equal signs are present, the first one is the delimiter between the
key and value while subsequent ones are part of the value.
2. A valid key-value pair cannot have an empty key or value.
3. The hash sign `#` starts a comment.
4. Any line not conforming to the above definition is ignored and can be used
as comment.
5. If a key appears more than once in the file, its last valid value is used.

In the configuration file the bracketed text (like `[Server]`) serves as
section titles.  Before version 0.4 they must be so.  Since version 0.4 they
are no different from a comment (by rule 4 above) and can be safely ignored or
removed.  It is recommended to use only the hash (`#`) sign to start a comment.


### Command-line options

All the options have equivalent command-line counterparts.  Therefore the
program can be used in the absence of a valid configuration file.  The
following options are recognized:

* `-u USERNAME`: Specifies the username.  Equivalent to the `username` option
in the config file.
* `-p PASSWORD`: Specifies the password.  Equivalent to the `password` option
in the config file
* `-a ADDRESS`: The host name or IP address of the server.  Equivalent to
the `address` option in the config file.
* `-P PORT`: Service port number.  Equivalent to the `port` option in the
config file.
* `-i INTERFACE`: Network interface userd for MAC address retrieval.
Equivalent to the `interface` option in the config file.
* `-s FILE`: Path to the "uid" storage file.  Equivalent to the `uidfile`
option in the config file.
* `-l FILE`: Path to the log file.  If the file doesn't exist, attempt to
create it.  If this option is not provided, no logs are kept.  If `-d` 
(debugging, see below) option is enabled, debugging information will be logged,
otherwise only certain exceptions are recorded.  Equivalent to the `logpath`
option in the config file.

*NOTE:*  The above command-line options, when specified, overrides the
equivalent configuration file options.

There are also extra options and switches special to the command line:

* `-c FILE`: Path to the configuration file.  This is useful when an
alternative path other than the default (`~/.pysrun.cfg`) is used.  Tip: use
`-c /dev/null` to get rid of a configuration file completely.
* `-I`: Turns on interactive password prompt.  The password you typed is not
echoed back.  This is useful if you don't want to store the password.  If
present, only the interactively typed password is used.
* `-d`: Turns on debugging output.  If logging (see above) is enabled, debug
information will be logged.
* `-h`: Display usage help and exit.


### Log-in, log-out and kicking operations.

1. Log-in: use the command `pysrun login`.  Program exits silently on
successful log-in.  The username and password are required.
2. Log-out: use `pysrun logout`.  The "uid" is required, therefore `uidfile`
must be known to the program.
3. Forced log-out, or kicking: use `pysrun kick` to force out all users of
this account.  The username and password are required.

*NOTE:* the executable file `pysrun` should be present in your `PATH`, otherwise
you have to use a fully qualified path to the `pysrun` executable.


### Exit code of the main program

* 0: Program exited successfully.  However, unexpected error may still happen.
For example, log-in can succeed but the program may fail to write the returned
uid to the specific location.  In this case a warning message will be sent
to stderr with the UID echoed.
* 1: The operation failed and server returned a message on the reason of
failure.  This is usually a result of reaching user numer limit, lack of
payment, or trying to log out while alreadly logged-out.
* 2: Program failed due to misconfiguration.
* 4: Operating system error, such as unreachable network, failure to read
from a file, or a specified network interface has no valid MAC address
information.
* 8: Malformed answer is received from the server, and the actual result is
unknown or undefined.


### Background process

After successful login, two daemon processes will run in background.  One
of them waits on further `login`, `logout` or `kick` commands, and
the other sends the server a heartbeat packet every one minute, keeping
the connection alive. This is a new feature absent in the old Linux client. 
After a new login, logout or kick operation the daemons will receive a command
to shut down, and will do so immediately.  They will also exit in one minute
after another user does a kick operation successfully, or the connection
becomes otherwise unavailable.  (Keeping the connection alive after being
kicked is useless.)  The daemons monitor each other, and one will immediately
exit if the other dies.

The daemons exit with code 0 on normal termination, and a non-zero status
otherwise.

During normal operation the monitor daemon opens a UNIX domain socket
at `/tmp/pysrun.socket`.  Undefined behavior would occur if the file is
made unavailable during the daemon's lifetime.


### Examples

1. `pysrun -c /dev/null -i wlan0 -u USERNAME -p PASSWORD login`  
Log in using the MAC address of interface wlan0 and no configuration file
(therefore, only the defaults).
2. `echo 'pysrun kick' | at midnight`  
Use the [`at`][at-man] command to submit a scheduled task: forcing all users
offline at 00:00.
3. Immediately after log-out or kicking, sometimes the server may say you are
already logged-in.  When this happens, just wait a moment and try
again, or use a command like  
`until pysrun login; do sleep 1; done`  
to automatically retry, if you are sure this is the *only* cause of failure.
If you don't want to see the spurious error messages, redirect them to
`/dev/null`:  
`until pysrun login 2> /dev/null; do sleep 1; done`  
Even better is to actually check the return code.  The following one-liner will
kick and try log in back immediately, with possible retries:  
``pysrun kick && while [[ `pysrun login 2> /dev/null; echo $?` -eq 1 ]]; do sleep 1; done``


## SYSTEM REQUIREMENTS

This program has been tested to work in Linux.  It is made to, and should, work
with BSD variants (including Free/Net/Open/DragonFlyBSD) but remains untested
for now.

It does not depend on whether the system is 32- or 64-bit.

Python version 2.7 is recommended.


## BACKGROUND INFORMATION

The so-called "official" Linux client provided by BNU is a dynamically-linked
32-bit executable.  It is defective, for these reasons:

* No 64-bit support.  You need to install the 32-bit libc just for it to work.
* Is a black-box binary and no one knows why or how it works or does not work.
* Has no meaningful way to configure, and does not return userful exit code.
Therefor it is almost impossible to be scripted.
* Since the Linux kernel 2.6.35 update, some reports the crash of the official
client.  This is because the official client uses a bad `bind()` call which
is blocked by the new kernel (see https://lkml.org/lkml/2011/7/9/37 or
[Linux commit d0733d2e][d0733d2e]).  So on newer systems the official client
fails to work at all.

> Note: on even newer Linux versions, `bind()`ing to an address of class
`AF_UNSPEC` is again allowed if the address is `INADDR_ANY`,
see [Linux commit 29c486df][29c486df].  This happens to be what the official
client does, so it happens to work again.  Admittedly this renders our last
argument moot.

Meanwhile, BNU has provided no support for BSD systems.  Even if many BSD
flavors now provide a compatibility layer for Linux binaries, one may expect
problems when executing the inherently sloppy official client.  A native
solution is still preferred.

Therefore, what we need is a new client free of those defects.  My new
client offers native support on Linux and BSD systems, is agnostic of the
"64-bittedness" of the OS, has open code and is not a sloppy work.  As a
result it is recommended you replace the "official" one by this new client.


## BUGS

Report bugs using the GitHub issue tracker: https://github.com/congma/pysrun/issues


## LICENSING

The programs is available under a BSD license.  See the packaged file COPYRIGHT.


## VERSION INFORMATION

2013-03-07 version 1.0.4.


[d0733d2e]: https://github.com/torvalds/linux/commit/d0733d2e29b652b2e7b1438ececa732e4eed98eb "Linux commit d0733d2e29b652b2e7b1438ececa732e4eed98eb"
[29c486df]: https://github.com/torvalds/linux/commit/29c486df6a208432b370bd4be99ae1369ede28d8 "Linux commit 29c486df6a208432b370bd4be99ae1369ede28d8"
[at-man]: http://linux.die.net/man/1/at "Linux manual page for at(1)"

