<!-- README.md -->

# sh-nspawn
#### A Linux container runtime written in Bash

sh-nspawn is a Linux container runtime written in bash.  It uses Linux
namespaces for isolation, a constructed tmpfs for `/dev`, veth interfaces, and a
socat allocated PTY device.  It aims to be as simple as possible, using only
GNU coreutils, iproute2, util-linux, and socat.

## Usage

    Usage: ./sh-nspawn -p <root> -b <bridge> -a <ip> -r <route>
      Start the operating system container at path <root>. Create a veth pair with
      the IPv4 address <ip> attached to the bridge device <bridge>, add a default
      route via <route>.

See [requirements](#requirements) for more details on host OS requirements.  See
[considerations](#considerations) for more details on container OS
considerations.

See the [hold my hand, please](#hold-my-hand-please) section for extact details
for setup.

## Requirements

The host requirements are intentionally kept simple; however, they require newer
versions of some software.  Although sh-nspawn has only been tested on Debian
stretch, it is expected to work so long as the following requirements are met:

* socat, at least version 1.7.3.0
  * `rawer` termios option added (see
    [CHANGES](http://www.dest-unreach.org/socat/doc/CHANGES)).
* iproute2, at least version 3.19.0
  * "Allow to easy change network namespace" (see commit
    [52700d4](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/commit/?id=52700d40a2b3ee20afcdfa999c611fef0107579a)).
* util-linux, at least version 2.24-rc1
  * "unshare: add --fork options for pid namespaces" (see commit
    [5088ec3](https://git.kernel.org/pub/scm/utils/util-linux/util-linux.git/commit/?id=5088ec338fe5dcd7e9a2d8daf7e7fa7dd6f87c27)).
* Linux, at least version 4.6
  * Compiled with `CONFIG_NAMESPACES`, `CONFIG_SYSVIPC`, `CONFIG_IPC_NS`,
    `CONFIG_NET_NS`, `CONFIG_PID_NS`, `CONFIG_UTS`, `CONFIG_CGROUPS`,
    `CONFIG_VETH`, and `CONFIG_BRIDGE` (see
    [**namespaces**(7)](http://man7.org/linux/man-pages/man7/namespaces.7.html),
    [**clone**(2)](http://man7.org/linux/man-pages/man2/clone.2.html),
    [**svipc**(7)](http://man7.org/linux/man-pages/man7/svipc.7.html)).

You'll need to setup a bridge interface to bridge the containers veth
interface to.  No need to setup DHCP on this interface, IP addresses and routes
must be static.

## Considerations

If the container ever calls
[**vhangup**(2)](http://man7.org/linux/man-pages/man2/vhangup.2.html) on the PTY
`/dev/console` device, `socat` will exit, the container's init process will
become a child of the host's init process, and you won't be able to reach it
from the console ever again.  You will likely have to remove the `TTYVHangup`
directive from the `getty@.service` (or similar).

## Limitations
As sh-nspawn is designed for educational purposes and is not intended to be used
in production, it's device support is very limited. For example:

* It only supports init binaries located at `/sbin/init` relative to the
  container's root.
* It does not leverage Linux user namespaces for "unprivileged" containers.
* It only creates one veth interface (`eth0`) which must be bridged on the host.
* It only allocates one PTY, which `/dev/console` is linked to.
* It only creates a minimal set of device nodes:
  * `/dev/tty` and `/dev/console`
  * `/dev/ptmx`, `/dev/pts/ptmx`, and `/dev/pts/0`
  * `/dev/null`, `/dev/zero`, and `/dev/full`.
  * `/dev/random` and `/dev/urandom`

## Hold my hand, please

*Run all commands as root.*

First, you'll need to install the pre-requisites on Debian stretch (or better).
Run `apt-get install socat bash coreutils util-linux socat`.

You will need a container root filesystem, you can create one on Debian-like
distributions using the `debootstrap` command from the `debootstrap` package.
You will have to create a directory for your new container `mkdir -p
/var/lib/containers/myct`.  Then you can install Debian into the directory
`debootstrap /var/lib/containers/myct stretch` on Debian, or `debootstrap
/var/lib/containers/myct xenial` on Ubuntu.

You'll need to create your own getty service file so that systemd doesn't
hang-up your PTY terminal or require a "real" tty.  Copy the current one to
a new path `cp /var/lib/containers/myct/lib/systemd/system/getty@.service
/var/lib/containers/myct/etc/systemd/system/getty-nohangup@.service`.  Edit the
new `getty-nohangup@.service` with your favorite editor, comment out the
`ConditionPathExists=/dev/tty0`, `TTYVHangup=yes`, and `TTYVTDisallocate=yes`
lines.  Enable it for `console` with `chroot /var/lib/containers/myct systemctl
enable getty-nohangup@console.service`.

In order to log into your new container you'll need to set a root password for
your container with `chroot /var/lib/containers/myct passwd`.  You'll also need
to ensure the`pts/0` is in the containers `securetty` file at
`/var/lib/containers/myct/etc/securetty`.

You must create a bridge interface to attach a veth interface too.  You can add
a bridge named `mybr0` with iproute2 using `ip link add name mybr0 type
bridge`, or with brctl using `brctl addbr mybr0`.  There are several ways to
provide network access via this bridge, but the safest is to route it with NAT
masquerading.  To do so you need to give your bridge an IP address, for example
`ip addr add 172.34.44.1/24 dev mybr0`.  You will need to enable IP forwarding
`sysctl -w net/ipv4/ip_forward=1` and enable NAT `iptables -I POSTROUTING -s
172.34.44.0/24 \! -o mybr0 -j MASQUERADE` (**note**: this has network security
implications).  This is the bridge interface you will provide to the `-b`
option; the IP you've chosen for your bride must be passed to the `-r`.  You'll
then need to select an unused IP address on the subnet you've chosen and use it
for the `-a` option.

Finally, start the container with `./sh-nspawn -p /var/lib/containers/myct -b
mybr0 -a 172.34.44.2/24 -r 172.34.44.1`, you should be able to log into the
console with the password you set.

## License

    sh-nspawn - A Linux container runtime written in Bash
    Copyright (C) 2018  Isabell Cowan

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.

<!-- vim: set ts=2 sw=2 et syn=markdown: -->
