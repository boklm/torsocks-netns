This repository contains torsocks-netns, a small wrapper script for
torsocks which creates an empty network namespace to run torsocks inside
it, blocking all connections except to the tor socks port.

This is a quick prototype for a network-namespace-based torsocks:
* https://gitlab.torproject.org/tpo/community/hackweek
* https://pad.riseup.net/p/2021-hackweek-network-namespace


How it works
------------

This program will:

 * run `socat` to listen on a UNIX socket in `$tmp_dir/torsocks.sock` and
   connect it with `localhost:9050`
 * create a user and network namespace
 * run `socat` inside the new namespace to connect `localhost:9050` with
   the UNIX socket in `$tmp_dir/torsocks.sock`
 * run the selected command with `torsocks` inside the new namespace


When using the --TransPort option it will work differently:

 * create a user and network namespace
 * use slirp4netns to add a new network device to the new network
   namespace and enable networking
 * set some iptable rules to redirect all connections to Tor (this part
   is still missing)
 * run the selected command inside the new namespace (without using torsocks)


Dependencies
------------

If you are using Debian, the following packages need to be installed:
 * socat
 * uidmap
 * libpath-tiny-perl

You can use this command:

  apt install socat uidmap libpath-tiny-perl

With the Debian kernel the `user_namespaces(7)` are disabled by default.
You can enable them with the following command as root:

  sysctl -w kernel.unprivileged_userns_clone=1

If using the --TransPort option you will also need the slirp4netns
package.


Usage
-----

Usage of this script is:

<pre>
  torsocks-netns [OPTIONS] -- [TORSOCKS-OPTIONS] [COMMAND [ARG...]]

  Options:
    --help
      Print this message.

    --port=&lt;port&gt;
      Set Tor port (default: 9050).
</pre>


Examples
--------

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
Saving to: ‘index.html’

index.html                                 100%[=====================================================================================>]  17.95K  --.-KB/s    in 0.08s   

2021-03-29 19:52:06 (231 KB/s) - ‘index.html’ saved [18381/18381]
</pre>

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
Saving to: ‘index.html’

index.html                                 100%[=====================================================================================>]  17.95K  --.-KB/s    in 0.05s   

2021-03-29 19:54:50 (348 KB/s) - ‘index.html’ saved [18381/18381]

# unset LD_PRELOAD
# wget https://torproject.org/
--2021-03-29 19:54:54--  https://torproject.org/
Resolving torproject.org (torproject.org)... failed: Temporary failure in name resolution.
wget: unable to resolve host address ‘torproject.org’
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


No Rights Reserved
------------------

Files in this repository are public domain (or CC0, see COPYING file).
