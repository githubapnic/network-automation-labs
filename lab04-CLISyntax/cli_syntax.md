![](images/apnic_logo.png)
# LAB: The Salt CLI Syntax: Targeting

For this lab, there's a Proxy Minion started for every device in the topology. As a reminder, the topology can be found 
below:

![](images/topology.png)

# LAB: Logging into the new system
There may be an issue with a conflict with the fingerprint used for Lab01 to Lab03. 

<pre>
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:9+JSueZJQBYdLyT9L7QBsaTexOnWBtokH3i0TWb8CIc.
Please contact your system administrator.
Add correct host key in /home/apnic/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/apnic/.ssh/known_hosts:56
  remove with:
  ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "groupXX.labs.apnictraining.net"
Host key for group105.labs.apnictraining.net has changed and you have requested strict checking.
Host key verification failed.
</pre>

The easiest fix is to delete the old key from the known_hosts file.

<pre>
ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "groupXX.labs.apnictraining.net"
</pre>

Otherwise, to bypass this check you can try this command:

```bash
ssh -o "StrictHostKeyChecking no" -i salt-lab.key root@groupXX.labs.apnictraining.net
```

Replace groupXX with the allocated group.

## Part-1: Targeting using Minion ID

Execute commands against individual devices, using the device name / Minion ID:

```bash
salt router1 test.ping
```

<pre>
root@salt:~# salt router1 test.ping
router1:
    True
</pre>


```bash
salt spine1 grains.get version
```

<pre>
root@salt:~# salt spine1 grains.get version
spine1:
    4.30.3M-33434233.4303M
</pre>

```bash
salt leaf1 pillar.get proxy
```

<pre>
root@salt:~# salt leaf1 pillar.get proxy
leaf1:
    ----------
    driver:
        ios
    host:
        leaf1
    password:
        APNIC2021
    proxytype:
        napalm
    username:
        apnic
</pre>

## Part-2: Targeting against a list of devices

Given a list of known devices, using the name / Minion ID:

```bash
salt -L 'router1,router2' test.ping
```

<pre>
root@salt:~# salt -L 'router1,router2' test.ping
router1:
    True
router2:
    True
</pre>

**HINT**: Use salt commands help guide, to view what the optional *-L* switch and other swicthes can do:

```bash
salt /? | grep -A 2 "\-L"
```

<pre>
root@salt:~# salt /? | grep -A 2 "\-L"

Incomplete options passed.

    -L, --list          Instead of using shell globs to evaluate the target
                        servers, take a comma or whitespace delimited list of
                        servers.
</pre>

Try this command to see more options.

```bash
salt --help | grep -E -A 2 "\-L|\-E|\-G|\-P|\-I|\-J"
```

## Part-3: Shell-like globbing

Execute commands against a globbing expression to match multiple devices:

```bash
salt 'leaf*' grains.get serial
```

<pre>
root@salt:~# salt 'leaf*' grains.get serial
leaf3:
    9I8N4XHDSR3
leaf1:
    9XX8EUA92Y6
leaf2:
    9Q78Z7176ZI
leaf4:
    9S1V3WVQ3OA
</pre>

```bash
salt 'spine[1,3]' grains.get version
```

```
root@salt:~# salt 'spine[1,3]' grains.get version
spine1:
    4.30.3M-33434233.4303M
spine3:
    4.30.3M-33434233.4303M
```
```bash
salt 'spine[!1,3]' grains.get version
```

```
root@salt:~# salt 'spine[!1,3]' grains.get version
spine2:
    4.30.3M-33434233.4303M
spine4:
    4.30.3M-33434233.4303M
```

## Part-4: Regular expressions

Target using regular expressions on the Minion ID

```bash
salt -E 'spine\d+' grains.get version
```

<pre>
root@salt:~# salt -E 'spine\d+' grains.get version
spine1:
    4.30.3M-33434233.4303M
spine3:
    4.30.3M-33434233.4303M
spine2:
    4.30.3M-33434233.4303M
spine4:
    4.30.3M-33434233.4303M
</pre>

## Part-5: Targeting using Grains

Reminder: Grains represents data collected by Salt and made available to you. Can be used to select groups of devices 
based on their properties:

```bash
salt -G vendor:Cisco grains.get os
```

<pre>
root@salt:~# salt -G vendor:Cisco grains.get os
leaf3:
    ios
leaf4:
    ios
leaf2:
    ios
leaf1:
    ios
core1:
    iosxr
core2:
    iosxr
</pre>

## Part-6: Targeting using Grains PCRE

Matching using regular expressions on Grains:

```bash
salt -P 'os:ios.*' net.cli 'show version | include Version'
```

<pre>
root@salt:~# salt -P 'os:ios.*' net.cli 'show version | include Version'
Executing job with jid 20210105181624808521
-------------------------------------------

leaf4:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XE Software, Version 03.15.00.S - Standard Support Release
            Cisco IOS Software, CSR1000V Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(2)S, RELEASE SOFTWARE (fc3)
            licensed under the GNU General Public License ("GPL") Version 2.0.  The
            software code licensed under GPL Version 2.0 is free software that comes
            GPL code under the terms of GPL Version 2.0.  For more details, see the
    result:
        True
core1:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XR Software, Version 6.0.0[Default]
            ROM: GRUB, Version 1.99(0), DEV RELEASE
    result:
        True
leaf1:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XE Software, Version 03.15.00.S - Standard Support Release
            Cisco IOS Software, CSR1000V Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(2)S, RELEASE SOFTWARE (fc3)
            licensed under the GNU General Public License ("GPL") Version 2.0.  The
            software code licensed under GPL Version 2.0 is free software that comes
            GPL code under the terms of GPL Version 2.0.  For more details, see the
    result:
        True
core2:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XR Software, Version 6.0.0[Default]
            ROM: GRUB, Version 1.99(0), DEV RELEASE
    result:
        True
leaf3:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XE Software, Version 03.15.00.S - Standard Support Release
            Cisco IOS Software, CSR1000V Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(2)S, RELEASE SOFTWARE (fc3)
            licensed under the GNU General Public License ("GPL") Version 2.0.  The
            software code licensed under GPL Version 2.0 is free software that comes
            GPL code under the terms of GPL Version 2.0.  For more details, see the
    result:
        True
leaf2:
    ----------
    comment:
    out:
        ----------
        show version | include Version:
            Cisco IOS XE Software, Version 03.15.00.S - Standard Support Release
            Cisco IOS Software, CSR1000V Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(2)S, RELEASE SOFTWARE (fc3)
            licensed under the GNU General Public License ("GPL") Version 2.0.  The
            software code licensed under GPL Version 2.0 is free software that comes
            GPL code under the terms of GPL Version 2.0.  For more details, see the
    result:
        True
</pre>


## Part-7: Targeting using Pillar

```bash
salt -I proxy:driver:iosxr net.load_config text='ntp server 10.0.0.1'
```

<pre>
root@salt:~# salt -I proxy:driver:iosxr net.load_config text='ntp server 10.0.0.1'
core2:
    ----------
    already_configured:
        False
    comment:
    diff:
        --- 
        +++ 
        @@ -1,6 +1,9 @@
         !! Last configuration change at Tue Jan  5 18:13:23 2021 by apnic
         !
         hostname core2
        +ntp
        + server 10.0.0.1
        +!
         interface MgmtEth0/0/CPU0/0
          ipv4 address 10.0.0.15 255.255.255.0
         !
    loaded_config:
    result:
        True
core1:
    ----------
    already_configured:
        False
    comment:
    diff:
        --- 
        +++ 
        @@ -1,6 +1,9 @@
         !! Last configuration change at Tue Jan  5 18:13:23 2021 by apnic
         !
         hostname core1
        +ntp
        + server 10.0.0.1
        +!
         interface Loopback0
         !
         interface MgmtEth0/0/CPU0/0
    loaded_config:
    result:
        True
</pre>

## Part-8: Targeting using Pillar PCRE

Apply regular expression on Pillar data:

```bash
salt -J 'proxy:host:spine[1|2]' net.arp --out=yaml
```

<pre>
root@salt:~# salt -J 'proxy:host:spine[1|2]' net.arp --out=yaml
spine1:
  spine2:
  comment: ''
  out:
  - age: 3753.0
    interface: Ethernet1/1
    ip: 10.23.12.0
    mac: 0C:00:D1:02:E9:05
  - age: 3705.0
    interface: Ethernet1/2
    ip: 10.23.22.0
    mac: 0C:00:24:19:BB:06
  - age: 3755.0
    interface: Ethernet1/3
    ip: 10.34.21.1
    mac: 0C:00:EC:61:4A:02
  - age: 3759.0
    interface: Ethernet1/4
    ip: 10.34.22.1
    mac: 0C:00:B5:20:24:01
  - age: 3761.0
    interface: Ethernet1/5
    ip: 10.34.24.1
    mac: 0C:00:C7:2F:2A:04
  - age: 3752.0
    interface: Ethernet1/6
    ip: 10.34.23.1
    mac: 0C:00:C9:40:32:04
  - age: 0.0
    interface: Management1
    ip: 172.22.0.10
    mac: 02:42:AC:16:00:0A
  result: true
spine1:
  comment: ''
  out:
  - age: 3802.0
    interface: Ethernet1/1
    ip: 10.23.11.0
    mac: 0C:00:45:0A:51:04
  - age: 3699.0
    interface: Ethernet1/2
    ip: 10.23.21.0
    mac: 0C:00:8C:DA:4A:07
  - age: 3803.0
    interface: Ethernet1/3
    ip: 10.34.12.1
    mac: 0C:00:E6:D6:02:02
  - age: 3806.0
    interface: Ethernet1/4
    ip: 10.34.11.1
    mac: 0C:00:65:B2:9C:01
  - age: 3798.0
    interface: Ethernet1/5
    ip: 10.34.14.1
    mac: 0C:00:06:52:6F:03
  - age: 3799.0
    interface: Ethernet1/6
    ip: 10.34.13.1
    mac: 0C:00:9A:97:B3:03
  - age: 0.0
    interface: Management1
    ip: 172.22.0.9
    mac: 02:42:AC:16:00:09
  result: true
</pre>

---
**End of Lab**

---
