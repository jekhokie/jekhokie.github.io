---
layout: post
title:  "Limits - The Silent Killers"
date:   2016-09-13 21:16:04 -0400
categories: ubuntu ulimit sockets tcp jenkins
---
Some troubleshooting techniques when seeing strange behavior on an Ubuntu system related to stalled
connections through a web request, terminated processes, etc. This post focuses specifically on
open file limits and number of allowable TCP sockets.

### File Limits

#### General

Running processes natively have their available resources restricted by the underlying operating
system. There are two types of limits - soft and hard. The default system limits can be seen via
running the following command:

```bash
$ sudo ulimit -aH
$ sudo ulimit -aS
```

#### Soft vs. Hard Limits

Soft limits (designated via `-aS`) are boundaries that, if breached, cause a SIGX signal to be sent
to the running process indicating it must lower its consumption immediately. This process can be
caught by the process (and typically should be) to handle adjustment of its consumption of resources
so that it is within the bounds of the specified limits.

Hard limits (designated via `-aH`) on the other hand, are final upper bounds for each respective
limit type. These values are the absolute maximum that a process can reach before the operating
system issues a SIGKILL signal to the process. This signal cannot be caught and will immediately
terminate the process.

#### Limit Inspection

A process can adjust its own hard limit down below the default hard limit, but never above. It can
likewise adjust its own soft limit down or up, but never above its own specified hard limit.

Running the `ulimit` command lists the file limits for the current user/session. In order to see
the limits for a running process (which may have been adjusted as part of the start script/launch
procedure), you can inspect the process `limits` file:

```bash
$ sudo cat /proc/<PID>/limits
```

Inspection of limits varies by limit type - one more common inspection has to do with running out of
file descriptors. Running the `cat` command above will produce output, one data point being `Max open
files`. This is the total number of allowable open file descriptors. To see how many file descriptors
a process currently has open (to inspect whether it is within range of the limit):

```bash
$ sudo ls -l /proc/<PID>/fd/ | wc -l
```

This will count the number of files in the `fd/` directory for the process, which is the total number
of open file handles the process has. Compare this to the number of files limit to inspect whether
the process is within range of its limit.

#### Limit Adjustment

Adjusting limits for a user (or all users) can be done in several ways, each of which having their own
long-term impact/effect. In general, to set a limit for a user, edit the following file like so:

```bash
$ sudo vim /etc/security/limits.conf
# limits are specified as <user> <limit_type> <limit_type> <limit>
# for example, to limit the hard and soft max number of files for the abc user to 1024:
#   abc soft nofile 1024
#   abc hard nofile 1024
```

Once the above are specified, to ensure that new sessions for the abc user respect the limits specified,
edit the following file/specify the following:

```bash
$ sudo vim /etc/pam.d/common-session
# placing this line in the file ensures all new session will require loading the limits.conf file
# that was configured in the step above
#   session required pam_limits.so
```

The above commands work for persisting limits moving forward. However, for already-running processes,
these files have no effect/impact. There is, however, a tool that can be used (`prlimit`) that can
adjust limits for running processes. To use this tool (as an example):

```bash
# adjust the number of files limit for process 12345 to soft=4096,hard=8192
$ sudo prlimit --pid 12345 --nofile=4096:8192
```

### TCP Connections (Sockets)

When a system opens a TCP connection to an endpoint, a socket is consumed. Sockets are also considered
files from a Unix system perspective, so one item to note is that if you are experiencing connection
issues and you suspect it may be a socket limit, you may first wish to investigate the number of files
limit for the process (see above section).

#### Ephemeral Port Range

The ports allowed by a system to be utilized/consumed by processes are defined in the sysctl settings
for the system. To inspect the available ports for use, the following command is useful:

```bash
# investigate ephemeral port range that is allowed by system
$ sudo sysctl net.ipv4.ip_local_port_range
# this will output something like the following:
#   net.ipv4.ip_local_port_range = 32768 61000
```

The above command (with sample output) means that ports 23768 through 61000 are allowed for use by
the system processes.

#### Socket Inspection

To investigate the number of sockets used on a particular system, there are several methods available:

```bash
# inspect total sockets for a system
$ sudo netstat -a | wc -l
$ sudo lsof -i | wc -l
$ sudo ss -s | grep Total:

# inspect total TCP sockets for a particular process ID
$ sudo cat /proc/<PID>/net/tcp | wc -l
$ sudo netstat --all --proc | grep <PID> | wc -l
$ sudo lsof -i -P | grep <PID> | wc -l
$ sudo cat /proc/2611/net/sockstat | grep sockets
```

#### Troubleshooting Socket Exhaustion

One common issue with processes that execute a high number of concurrent connections is socket
exhaustion. In this case, the total number of sockets used by the processes on a system equals
the total available Ephemeral Ports (defined in the section above). In this situation, the system
will effectively block new connections until an available socket is released by an existing process.

##### TIME_WAIT

Although highly-performing systems are typically tuned for their TCP settings, in some cases socket
starvation may occur due to sockets being left around in the TIME_WAIT state. This state defines one of
the last states a socket remains in prior to completely closing/being released and exists to ensure out
of order packets are handled appropriately. While in the TIME_WAIT state, the socket is effectively 'in
use' according to the system (counts against the total number of used sockets that are unavailable to
existing processes). By default, a socket will remain in TIME_WAIT for 120 seconds (2 minutes) without
any existing tuning of the system.

To detect if there are a lot of sockets in the TIME_WAIT state, the following commands are useful:

```bash
$ sudo netstat -a | grep TIME_WAIT
$ sudo ss -a | grep TIME_WAIT
```

If socket exhaustion occurs and there are a lot of sockets in the TIME_WAIT state, it may be useful to
adjust the time a socket is allowed to exist in TIME_WAIT prior to being re-used by the system. This is
useful where network connectivity is fast/reliable and out of order packets/late packets are unlikely.
To tune the used sockets in TIME_WAIT, there are a couple useful things you can do.

To decrease the timeout (time to close) a TIME_WAIT socket:

```bash
$ sudo echo <TIMEOUT_SEC> > /proc/sys/net/ipv4/tcp_fin_timeout
```

To instruct the system to re-use sockets that are in TIME_WAIT when socket exhaustion has occurred,
enable the `tcp_tw_reuse` parameter. This will ensure that if all available sockets NOT in the TIME_WAIT
state are consumed, the system will start to reap the TIME_WAIT sockets/allocate them to processes that
need them:

```bash
$ sudo echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
```

Following specifying the re-use parameter, if performance is still not good enough, it may be desirable
to enable another setting which tells the system to re-allocate TIME_WAIT sockets as soon as the next socket
is requested (instead of allocating available sockets to a process). This is a bit more heavy of an
implementation but may still be useful:

```bash
$ sudo echo 1 > /proc/sys/net/ipv4/tcp_tw_recycle
```

##### Increase Available Sockets

As stated earlier in this post, the total number of sockets available for allocation is defined via the
`ip_local_port_range` parameter in the sysctl.conf file. For high-connection systems, the default Unix
setting for this is typically a bit lower than desirable. Many people will adjust this parameter to
increase the total available sockets via the following (thus increasing the total connection count
available to the system). However, it's worth noting that it is likely a NOFILE (number of files) hard
limit would be hit prior to experiencing complete socket starvation due to this setting, so it is worth
investigating the file limits and file descriptors consumed prior to exploring this setting.

```bash
# adjust the available port range for the system
$ sudo sysctl -w net.ipv4.ip_local_port_range="1025 65535"

# ensure range persists on reboot
$ sudo vim /etc/sysctl.conf
# add the following:
#   net.ipv4.ip_local_port_range = 1025 65535
```
