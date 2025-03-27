# mdevd-as-an-admin

A replacement for [sys-fs/udev][1] or [sys-apps/systemd-utils[udev]][1] using skarnet's [mdevd][2] and [execline][3]. Manages networks, filesystems, USB device authentication, and power events, without [dbus][4], [systemd][1], [pam][5], or [logind][6].

The base objectives of this project are to remove all dependencies on the above cruft (systemd&friends), and to improve security by reducing complexity to auditable levels. To ensure maximal user control, all code is simple scripting.  These scripts are written in [execline][3] to remove the complexities of auditing the security of a shell interpreter. The user can easily replace them with his own shell scripts, if desired.

To use this code, most people will need to be very hands-on in their system administration, and adjust code to their needs.  However, specialized complete distributions using this code may be released for specific targets.

## `/etc/mdevd.conf`

[mdevd][2] configuration is in `/etc/mdevd.conf` (note that mdevd defaults to `/etc/mdev.conf`, either rename it, or reference it when invoking mdevd)  This file references multiple execline scripts located in `/usr/lib/mdevd/`.

These scripts will also maintain `/dev/\*/by-\*/\*` symlinks, `/etc/fstab` entries, and mountpoints at `/media/*` for those entries. By default these mountpoints can then be mounted and unmounted by unprivileged users.

## `/etc/mactab`

When network devices are added, they are identified by their [MAC address][7], and configured according to an expanded interpretation of the `/etc/mactab` file used by `nameif` from [sys-apps/net-tools][8].  This expanded interpretation asserts that each interface configuration is defined by one line of text consisting of up to five fields separated by white space:

* 1: NAME: the new name you wish this interface to be known by.

* 2: ORIGINAL MAC: the factory MAC address of the interface.

* 3: NEW MAC: the [MAC address][7] you wish to apply to the interface.

* 4: IP: The IP address and mask to be assigned to the interface.

* 5: GATEWAY: The IP address of a default gateway

The third field: 'NEW MAC' may be any valid [MAC address][7], or one of the key words: 'same', or 'random'.  For 'random' to work, [net-analyzer/macchanger][8] must be installed.

The fourth field: IP may be any IP/mask e.g. `192.168.0.9/24`, or the key word `dhcpcd`. If the latter, [net-misc/dhcpcd][10] must be installed.  At present this merely uses `s6-svc` to bring up the service `/run/service/dhcpcd`. If you use something else, edit the `/usr/lib/mdevd/network-add` execline script.

The fifth field: 'GATEWAY' may be any valid IP address in the same network as the 'IP' field, or the keyword 'dhcpcd', which allows a fixed IP address to be assigned prior to getting an additional IP address, and gateway via dhcp.

Only the first two fields are mandatory, for backward compatibility with `nameif`.

## `/etc/usbauth`

When unauthorized USB devices are connected, a hopefully unique identifier string is constructed from device's attributes.  If this string occurs in the `/etc/usbauth` file, then the device will be authorized.  A simple 'grep' is used, so all text outside the string is considered 'comment'.  If not found, the constructed string will be printed to mdevd standard out, normally a log file.  To add authentication for a new device, copy the string from the log file to `/etc/usbauth`, and add comments to taste.

## `/usr/lib/mdevd/`

This is the directory where the execline scripts are located. They are:

* `input-add` and `input-remove`  
	These scripts manage symlinks in `/dev/`, and can invoke a local script unique to the input when the deviced is added.

* `network-device-add`  
	This script manages configuring and bringing up network interfaces, and can invoke a local script unique to each device.  This can be used, for example, to create a bridge from two interfaces, after they have been added, which thereby creates the bridge as a new interface, which can then be configured by `/etc/mactab`.

* `block-device-add` and `block-device-remove`  
	These scripts manage symlinks in `/dev/`, and for mountable filesystems, it will manage an entry in `/etc/fstab`, and a corresponding mount point at `/media/`. These mountpoints are mountable and unmountable by unprivileged users, unless you set permissions otherwise.  When an encrypted filesystem is encountered, if an instance of X is running, 'yad' will run on that DISPLAY as the user which X is running as, and ask for a password to decrypt the filesystem.  If X is not running the 'uevent' file associated with the device will be set group writable to permit members of that group to later re-invoke the decrypt attempt by writing 'add' to that file.

* `rfkill`  
	This script merely uses `s6-svc` to start and stop the `/run/service/iwd` service. (use [net-wireless/eiwd][11] to avoid [dbus][4])

* `power`  
	This script changes screen brightness and CPU governors when AC power is added or removed.

[1]: https://systemd.io/
[2]: https://skarnet.org/software/mdevd/
[3]: https://www.skarnet.org/software/execline/
[4]: https://www.freedesktop.org/wiki/Software/dbus/
[5]: https://github.com/linux-pam/linux-pam
[6]: https://github.com/elogind/elogind
[7]: https://en.wikipedia.org/wiki/MAC_address
[8]: https://net-tools.sourceforge.io/
[9]: https://github.com/alobbs/macchanger
[10]: https://roy.marples.name/projects/dhcpcd/
[11]: https://github.com/illiliti/eiwd
