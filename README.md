This repository contains `torsocks-netns`, a small wrapper script for
`torsocks` which creates an empty network namespace to run `torsocks`
inside it, blocking all connections except to the tor socks port.

This is a quick prototype for a network-namespace-based torsocks:
* https://gitlab.torproject.org/tpo/community/hackweek
* https://gitlab.torproject.org/tpo/community/hackweek/-/blob/main/network-namespace-based-torsocks.md


How it works
------------

`torsocks-netns` can work in 3 different modes.

In the `torsocks` mode (default) it will:

 * run `socat` to listen on a UNIX socket in `$tmp_dir/torsocks.sock` and
   connect it with `localhost:9050`
 * create a user and network namespace
 * run `socat` inside the new namespace to connect `localhost:9050` with
   the UNIX socket in `$tmp_dir/torsocks.sock`
 * run the selected command with `torsocks` inside the new namespace

In the `redsocks` mode it will:

 * run `socat` to listen on a UNIX socket in `$tmp_dir/torsocks.sock` and
   connect it with `localhost:9050`
 * if the `--DNSPort` option is used, run `socat` to listent on a UNIX
   socket in `$tmp_dir/DNSPort.sock` and connect with `localhost:$DNSPort`
   (in udp).
 * create a user and network namespace
 * run `socat` inside the new namespace to connect `localhost:9050` with
   the UNIX socket in `$tmp_dir/torsocks.sock`
 * if the --DNSPort option is used, run `socat` inside the new namespace
   to connect `localhost:53` (udp) with the UNIX socket in
   `$tmp_dir/DNSPort.sock`
 * run `redsocks` and set some iptables rules to redirect all connections
   to redsocks, which is configured to redirect connections to the socks
   proxy on `localhost:9050`. If the `--DNSPort` option is used we also
   redirect all udp requests on port 53 to `127.0.0.1:53`. If the
   `--DNSPort` option is not used, the DNS will not work.
 * run the selected command inside the new namespace (without using
   `torsocks`)

In the `slirp4netns` mode (not working yet) it will:

 * create a user and network namespace
 * use `slirp4netns` to add a new network device to the new network
   namespace and enable networking
 * set some iptable rules to redirect all connections to Tor. This part
   is still missing.
 * run the selected command inside the new namespace (without using `torsocks`)


Dependencies
------------

If you are using Debian, the following packages need to be installed:
 * socat
 * uidmap
 * libpath-tiny-perl
 * libfindbin-libs-perl

You can use this command:

<pre>
  # apt install socat uidmap libpath-tiny-perl libfindbin-libs-perl
</pre>

With the Debian kernel the `user_namespaces(7)` are disabled by default.
You can enable them with the following command as root:

<pre>
  # sysctl -w kernel.unprivileged_userns_clone=1
</pre>

If using the `slirp4netns` mode you will also need the `slirp4netns`
package.

If using the `redsocks` mode you will also need the `redsocks` package.


Usage
-----

Usage of this script is:

<pre>
  torsocks-netns [OPTIONS] -- [TORSOCKS-OPTIONS] [COMMAND [ARG...]]

Options:
  --help
    Print this message.

  --mode=&lt;torsocks|redsocks|slirp4netns&gt;
    Default mode is torsocks.

  --SocksPort=&lt;port&gt;
    Set Tor Socks port (default: 9050).

  --TransPort=&lt;port&gt;
    Set Tor transparent proxy port (TransPort in torrc). When this is set,
    instead of using torsocks we use some iptable rules to redirect all
    connections to Tor. This requires setting the --DNSPort option too.
    Using this option automatically selects the slirp4netns mode.

  --DNSPort=&lt;port&gt;
    Set Tor DNSPort. This can only be used in redsocks and slirp4netns modes.
</pre>


Examples
--------

Using the `torsocks` mode (default):

<pre>
$ ./torsocks-netns wget https://torproject.org/
--2021-03-29 19:52:04--  https://torproject.org/
Resolving torproject.org (torproject.org)... 95.216.163.36
Connecting to torproject.org (torproject.org)|95.216.163.36|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://www.torproject.org/ [following]
--2021-03-29 19:52:05--  https://www.torproject.org/
Resolving www.torproject.org (www.torproject.org)... 95.216.163.36
Connecting to www.torproject.org (www.torproject.org)|95.216.163.36|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18381 (18K) [text/html]
Saving to: ???index.html???

index.html                                 100%[=====================================================================================>]  17.95K  --.-KB/s    in 0.08s   

2021-03-29 19:52:06 (231 KB/s) - ???index.html??? saved [18381/18381]
</pre>

In the `torsocks` mode, if we unset `LD_PRELOAD`, connections are not working anymore:

<pre>
$ ./torsocks-netns bash
# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# wget https://torproject.org/
--2021-03-29 19:54:48--  https://torproject.org/
Resolving torproject.org (torproject.org)... 95.216.163.36
Connecting to torproject.org (torproject.org)|95.216.163.36|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://www.torproject.org/ [following]
--2021-03-29 19:54:49--  https://www.torproject.org/
Resolving www.torproject.org (www.torproject.org)... 95.216.163.36
Connecting to www.torproject.org (www.torproject.org)|95.216.163.36|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18381 (18K) [text/html]
Saving to: ???index.html???

index.html                                 100%[=====================================================================================>]  17.95K  --.-KB/s    in 0.05s   

2021-03-29 19:54:50 (348 KB/s) - ???index.html??? saved [18381/18381]

# unset LD_PRELOAD
# wget https://torproject.org/
--2021-03-29 19:54:54--  https://torproject.org/
Resolving torproject.org (torproject.org)... failed: Temporary failure in name resolution.
wget: unable to resolve host address ???torproject.org???
</pre>

In the `redsocks` mode, the `DNSPort` option should be added to the
`torrc` file, and the `--DNSPort` option used (if not, DNS won't be
working):
<pre>
$ ./torsocks-netns --DNSPort=9053 --mode=redsocks -- wget -q -O- https://check.torproject.org/ | grep Congratulations
      Congratulations. This browser is configured to use Tor.
      Congratulations. This browser is configured to use Tor.
</pre>

In the `redsocks` mode, `LD_PRELOAD` is not used:
<pre>
$ ./torsocks-netns --DNSPort=9053 --mode=redsocks -- bash
# unset LD_PRELOAD
# wget -q -O- https://check.torproject.org/ | grep Congratulations
      Congratulations. This browser is configured to use Tor.
      Congratulations. This browser is configured to use Tor.
</pre>


Known issues
------------

Because we use a user namespace, the program we run inside the user
namespace will think it is running as root. Maybe some programs will
not like that. The files owned by the same user are seen as owned by
root, and the files owned by other users are seen as owned by
nobody/nogroup. Unfortunately it seems it is not possible to create a
network namespace without creating at the same time a user namespace as
an unprivileged user.

The `slirp4netns` mode doesn't currently work: connections are not going
through Tor when using this mode.

If a program is using namespaces itself, it will maybe not work with
`torsocks-netns`, because of problems creating nested namespaces.


Posible improvements
--------------------

* The user's uid is currently mapped to the root uid inside the namespace.
  Maybe we can change the mapping so that the user's uid is matching
  inside and outside the namespace, so that it doesn't look like the
  program is running as root.

* In the `redsocks` mode, when the `--DNSPort` option is not used, there
  is no DNS working. Maybe we can find some way to make that work.

* The `slirp4netns` mode is not working currently. We are missing some
  iptables rules to redirect connections to Tor. We should remove this
  mode if we can't make it work.

* We can add a `--stream-isolate` option for `redsocks` mode, setting a
  random username and password for the SOCKS5 authentication (similar to
  torsocks' `-i`).


See also
--------

[orjail](https://github.com/orjail/orjail) is an other tool that seems
to be doing something similar, using firejail.


No Rights Reserved
------------------

Files in this repository are public domain (or CC0, see COPYING file).
