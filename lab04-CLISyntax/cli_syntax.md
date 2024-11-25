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
  ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "npnog10-vmXX.labs.apnictraining.net"
Host key for group105.labs.apnictraining.net has changed and you have requested strict checking.
Host key verification failed.
</pre>

The easiest fix is to delete the old key from the known_hosts file.

<pre>
ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "npnog10-vmXX.labs.apnictraining.net"
</pre>

Otherwise, to bypass this check you can try this command:

```bash
ssh -o "StrictHostKeyChecking no" -i salt-lab.key root@npnog10-vmXX.labs.apnictraining.net
```

Replace npnog10-vmXX with the allocated group.

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

**Note**: There may be a python error, but the command completes succesfully

<pre>
  root@salt:~# salt router1 test.ping
/usr/local/lib/python3.6/site-packages/requests/__init__.py:91: RequestsDependencyWarning: urllib3 (1.26.18) or chardet (3.0.4) doesn't match a supported version!
  RequestsDependencyWarning)
router1:
    True
</pre>

Add **2> /dev/null** to the end of the commands to suppress the python error. For example:

```bash
salt router1 test.ping 2> /dev/null
```

```bash
salt spine1 grains.get version
```

<pre>
root@salt:~# salt spine1 grains.get version
spine1:
    4.18.1F-4591672.4181F
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

<pre>
root@salt:~# salt 'spine[1,3]' grains.get version
spine3:
    4.18.1F-4591672.4181F
spine1:
    4.18.1F-4591672.4181F
root@salt:~# salt 'spine[!1,3]' grains.get version
spine2:
    4.18.1F-4591672.4181F
spine4:
    4.18.1F-4591672.4181F
</pre>

## Part-4: Regular expressions

Target using regular expressions on the Minion ID

```bash
salt -E 'spine\d+' grains.get version
```

<pre>
root@salt:~# salt -E 'spine\d+' grains.get version
spine2:
    4.18.1F-4591672.4181F
spine1:
    4.18.1F-4591672.4181F
spine4:
    4.18.1F-4591672.4181F
spine3:
    4.18.1F-4591672.4181F
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
salt -J 'proxy:host:spine[1|2]' net.mac --out=yaml
```

<pre>
root@salt:~# salt -J 'proxy:host:spine[1|2]' net.mac --out=yaml
spine1:
  comment: ''
  out:
  - active: true
    interface: Ethernet1
    last_move: 1609870868.989729
    mac: '52:54:00:06:24:02'
    moves: 1
    static: false
    vlan: 1
  - active: true
    interface: Ethernet7
    last_move: 1609870868.976326
    mac: 52:54:00:26:D7:01
    moves: 1
    static: false
    vlan: 1
  result: true
spine2:
  comment: ''
  out:
  - active: true
    interface: Ethernet7
    last_move: 1609870868.419326
    mac: '52:54:00:06:24:02'
    moves: 1
    static: false
    vlan: 1
  - active: true
    interface: Ethernet1
    last_move: 1609870868.432837
    mac: 52:54:00:26:D7:01
    moves: 1
    static: false
    vlan: 1
  result: true
</pre>

---
**End of Lab**

---
