# Ubuntu Server 20.04 LTS Setup Guide

Ubuntu 20.04 LTS will be supported until 2025 for freely available updates and up to 2030 for Extended Security Maintenance updates, a paid update service.


### Objectives

Briefly this guide covers:

* Establishing a baseline minimal install (removing unused software)
* Network and host firewall configuration
* Optional installation of common packages and package-specific configuration


### Base Install

This base install will have minimal packages and a footprint of about 10-15 GB on disk, recommended minimum OS volume is 20 GB.

*Note:* If the installer crashes soon after trying to create the partition table try to zero out the target disk using `dd if=/dev/zero of=/dev/sdX bs=1M count=100` on a second virtual terminal (`Alt+F2`) and then restart the installation process.


#### Set Timezone to UTC

Servers should always use UTC so that timestamps on logging are consistent across systems and timezones.  Use `dpkg-reconfigure tzdata` to set the time to UTC (under None of The Above category).


#### Removing cloud-init

By default Ubuntu 20.04 assumes it will be a cloud server and loads the cloud-init package.

To remove cloud-init: 

First use `dpkg-reconfigure cloud-init` and deselect all options except for none (the last option).

Then use `apt-get purge cloud-init` to remove the package and all supporting files.

(Optional) Clean-up folders with `rm -rf /etc/cloud /var/lib/cloud`


#### Removing open-iscsid

If iSCSI (remote storage) connections will not be needed, there is a service by default that will wait for networking and hang upon networking being unavailable.

This is the `iscsid.service`, it should be disabled for now (if we need iSCSI support we can re-enable it).

Verify using `systemctl show -p WantedBy network-online.target` and disable and remove using: 

```
systemctl disable open-iscsi.service
systemctl disable iscsid.service
apt-get remove open-iscsi
```


#### Remove netplan.io

```
apt-get remove netplan.io
systemctl enable systemd-networkd
```

Create network configuration files in `/etc/systemd/network` for each interface:

Default catch-all (enough to get things online):

```
# 99-default.network
[Match]
Name=en*

[Network]
DHCP=yes
```

And specifically for each interface, e.g. `eno1`.

```
# 10-eno1.network
[Match]
Name=eno1

[Network]
DHCP=yes
```

For bonding:

```
# 10-bond1.netdev
[NetDev]
Name=bond1
Kind=bond

[Bond]
Mode=802.3ad
TransmitHashPolicy=layer3+4
MIIMonitorSec=1s
LACPTransmitRate=fast
```

```
# 10-bond1.network
[Match]
Name=bond1

[Network]
DHCP=yes
BindCarrier=eno1 eno2
```

```
# 10-eno1.network
[Match]
Name=eno1

[Network]
Bond=bond1
```

```
# 10-eno2.network
[Match]
Name=eno2

[Network]
Bond=bond1
```

#### IPv6 support changes using DHCPv6

For network configuration we make use of systemd-networkd.  During setup IPv4 and IPv6 should be set to Automatic (IPv6 is disabled by default).

In 20.04 the default behavior is DUID Type 2 DUID-EN using an enterprise ID of `0x0000ab11` (43793) which is registered to the systemd project.

For Linux systems our preferred method for DUID generation is DUID Type 4, DUID-UUID.  

This should be generated using the system UUID in `/etc/machine-id` with a prefix of `0004`.  For example, a machine-id value of `8f05adf332ec47559d173618b17b1d99` would use a DUID of `00048f05adf332ec47559d173618b17b1d99`.

```
echo 0004`cat /etc/machine-id` | sed 's/../&:/g;s/:$//'
```

To configure the DUID type, edit `/etc/systemd/networkd.conf` and add


```
DUIDType=uuid
DUIDRawData=8f:05:ad:f3:32:ec:47:55:9d:17:36:18:b1:7b:1d:99
```

Replacing the DUID with the contents of `/etc/machine-id`

See: https://www.freedesktop.org/software/systemd/man/networkd.conf.html

Your DHCPv6 server can be configured to use the DUID to provide static addressing.


#### Remove ufw and install iptables/netfilter-persistent

Ubuntu makes use of UFW to manage firewall rules by default.  It’s recommended that UFW be removed in favor of `iptables-persistent` for server systems which may require non-trivial host firewall policy.

```
ufw disable
apt-get remove ufw
apt-get purge ufw
apt-get install netfilter-persistent iptables-persistent
```


#### Remove Snap

It's uncommon for a server to ever need support for snaps.

```
apt-get purge snapd
```


#### User Accounts and Remote Access

Install SSH

```
apt-get install ssh
```

Creating Administrator Account(s)

Use adduser to create a new user account:

```
adduser example
```

Then use usermod to append the sudo group to the new user:

```
usermod -aG sudo example
```

Verify that you’re able to login with the newly created user account, and gain a root shell using `sudo -i`, then disable the root account:

```
passwd -dl root
```

Add public SSH key to user account (recommended):

When logged in as the user:

```
mkdir .ssh
chmod 700 .ssh
nano .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

Paste the contents of your public SSH key into the `authorized_keys` file.  This will allow SSH key authentication for your account.  

Update your password to be a unique (not shared with any other system), strong (greater than 16 characters), password that is stored securely using a password manager.

If you make use of a emergency account that should not be allowed for remote access you can disable that user in SSH.  To do this update `/etc/ssh/sshd_config` and append the following line (this can be a space-separated list of users):

```
DenyUsers example
```

#### Host Firewall Configuration

After making any changes to the host firewall policy, be sure to save the configuration so it will persist after reboot.

```
service netfilter-persistent save
```

Example minimal host firewall policy for allowing SSH, HTTP, and HTTPS:

*IPv4*

```
iptables -P INPUT ACCEPT
iptables -F INPUT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -m recent --set --name DEFAULT --rsource
iptables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 --name DEFAULT --rsource -j DROP
iptables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -j DROP
```

*IPv6*

```
ip6tables -P INPUT ACCEPT
ip6tables -F INPUT
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 1 -j ACCEPT                      # Destination Unreachable [RFC4443]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 2 -j ACCEPT                      # Packet Too Big [RFC4443]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 3 -j ACCEPT                      # Time Exceeded [RFC4443]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 4 -j ACCEPT                      # Parameter Problem [RFC4443]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 128 -j ACCEPT                    # Echo Request [RFC4443]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 130 -j ACCEPT                    # Multicast Listener Query [RFC2710]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 131 -j ACCEPT                    # Multicast Listener Report [RFC2710]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 132 -j ACCEPT                    # Multicast Listener Done [RFC2710]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 134 -m hl --hl-eq 255 -j ACCEPT  # Router Advertisement [RFC4861]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 135 -m hl --hl-eq 255 -j ACCEPT  # Neighbor Solicitation [RFC4861]
ip6tables -A INPUT -p ipv6-icmp -m icmp6 --icmpv6-type 136 -m hl --hl-eq 255 -j ACCEPT  # Neighbor Advertisement [RFC4861]
ip6tables -A INPUT -p udp -m udp --sport 547 --dport 546 -m hl --hl-eq 255 -j ACCEPT    # DHCPv6
ip6tables -A INPUT -m conntrack --ctstate INVALID -j DROP
ip6tables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
ip6tables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -m recent --set --name DEFAULT --rsource
ip6tables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 --name DEFAULT --rsource -j DROP
ip6tables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
ip6tables -A INPUT -p tcp -m tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
ip6tables -A INPUT -p tcp -m tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT
ip6tables -A INPUT -j DROP
```

*Note 1:* The above policy makes use of the `recent` module for netfilter to limit SSH attempts to 3 new connections per minute in this example.  If you are using a script which makes many SSH connections quickly (such as rsync over SSH) you should create a rule to allow that specific host before the recent match to work around this connection limit.

*Note 2:* ICMPv6 filtering should be done with care.  The above policy is recommended as a baseline and makes use of the `hl` module (hop-limit) to enforce that permited ICMPv6 traffic is on-link and has not been routed.  This becomes important because valid ICMPv6 traffic can be sourced from either a link-local prefix or a global prefix by hosts which share a segment, so scoping ICMPv6 to only the link-local prefix will drop valid ICMPv6 and cause broken IPv6 connectivity in many cases.  The alternative would be creating prefix-specific rules but this is something we avoid when possible.


*Optional Outbound Host Firewall Policy*

The following outbound policy will restrict outgoing connections from the server to those run as root and DNS requests; additional examples may be exceptions for LDAP.  An outbound policy, even local, can often prevent attacks from being successful in downloading a larger application payload from a remote resource and is a good practice for hardening a system, but requires a good understanding of what processes and users will need to initiate outgoing connections.  Adopt with caution.

```
iptables -P OUTPUT ACCEPT
iptables -F OUTPUT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate INVALID -j DROP
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -p icmp -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate NEW -m owner --uid-owner root -j ACCEPT 
iptables -A OUTPUT -m conntrack --ctstate NEW -m owner --gid-owner root -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate NEW -p udp -m udp --dport 53 -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate NEW -p tcp -m tcp --dport 53 -j ACCEPT
iptables -A OUTPUT -j DROP
```

```
ip6tables -P OUTPUT ACCEPT
ip6tables -F OUTPUT
ip6tables -A OUTPUT -o lo -j ACCEPT
ip6tables -A OUTPUT -m conntrack --ctstate INVALID -j DROP
ip6tables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A OUTPUT -p ipv6-icmp -j ACCEPT
ip6tables -A OUTPUT -m conntrack --ctstate NEW -m owner --uid-owner root -j ACCEPT 
ip6tables -A OUTPUT -m conntrack --ctstate NEW -m owner --gid-owner root -j ACCEPT
ip6tables -A OUTPUT -m conntrack --ctstate NEW -p udp -m udp --dport 53 -j ACCEPT
ip6tables -A OUTPUT -m conntrack --ctstate NEW -p tcp -m tcp --dport 53 -j ACCEPT
ip6tables -A OUTPUT -j DROP
```

Example: Modify to allow apt-get updates (from 16.04 configuration notes, needs review):

```
iptables -I OUTPUT 9 -m conntrack --ctstate NEW -m owner --uid-owner _apt -j ACCEPT
ip6tables -I OUTPUT 9 -m conntrack --ctstate NEW -m owner --uid-owner _apt -j ACCEPT
```

Enable logging of output drops in syslog:

```
iptables -D OUTPUT -j DROP
iptables -A OUTPUT -j LOG --log-prefix "FW-OUTPUT " --log-level 6
iptables -A OUTPUT -j DROP
```

```
ip6tables -D OUTPUT -j DROP
ip6tables -A OUTPUT -j LOG --log-prefix "FW-OUTPUT " --log-level 6
ip6tables -A OUTPUT -j DROP
```


#### Package Maintenance

Remove unused packages:

```
apt-get autoremove
```

Remove any half-installed packages:

```
dpkg --purge $(dpkg --list | grep '^iU' | gawk '{print $2}' | xargs)
```

Remove any removed packages (deleting any leftover configuration files):

```
dpkg --purge $(dpkg --list | grep '^rc' | gawk '{print $2}' | xargs)
```


#### Time Server Configuration

Ubuntu 20.04 implements an NTP service as part of systemd-timesyncd replacing the older ntpd service.

Edit `/etc/systemd/timesyncd.conf` and update `NTP=` to the list of your desired time servers (separated by a space):

```
[Time]
NTP=ntp-1.example.com ntp-2.example.com
```

Restart the service:

```
systemctl restart systemd-timesyncd
```

Verify using:

```
timedatectl status
timedatectl timesync-status
timedatectl show-timesync --all
```


#### Upgrading from 18.04

List of changes to make after `do-release-upgrade`

*Fix MySQL:*

MySQL breaks on upgrade due to the presence of two lines in `/etc/mysql/my.cnf`

```
query_cache_limit    = 1M
query_cache_size     = 16M
```

Comment them out and restart the service:

```
## Broken from upgrade 18.04 to 20.04
## query_cache_limit    = 1M
## query_cache_size     = 16M
```

```
systemctl restart mysql.service
```

*Fixing Network Interface Names:*

It's good to have systems to be in similar configurations.  One of the drawbacks to `do-release-upgrade` is that the old form of network interface naming is preserved, making upgraded systems different from freshly installed systems.

To undo old interface names (e.g. eth0) the following files need to be removed:

```
rm /lib/udev/rules.d/71-biosdevname.rules
rm /etc/udev/rules.d/80-net-setup-link.rules
```

Then rebuild the initramfs and reboot:

```
update-initramfs -u
reboot
```

*WARNING:* Your network configuration will be broken and you will need console access to access the server if making this change.  You will need to update your network configuration to reflect corrected network interface names using the notes above.


