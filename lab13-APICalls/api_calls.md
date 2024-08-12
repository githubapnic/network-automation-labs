![](images/apnic_logo.png)
# LAB: Invoking low-level API functions

For this lab, we will have again all the Proxy Minions running, without any other external services. All the Minions are 
running using NAPALM for the connection.

Low-level API functions allow you to interact directly with the network device, and present you the data exactly in the 
way the device returns it. We've seen previously functions such as `net.arp`, `net.lldp`, and others; under the hood, 
these functions execute one or more low-level API calls, and eventually some data processing, in order to provide the 
data in a vendor-agnostic format, easy to consume. But these low-level API calls differ from one platform to another. In 
the next sections we will look into what Salt functions you can use per individual platform. In fact, we've already seen 
some of those in the previous lab, but now, we'll look into them closely.

## Part-1: Executing RPC calls on Juniper devices

RPC (Remote Procedure Call) is when a computer program causes a procedure (subroutine, or instruction) on another, 
remote machine. When talking about RPC calls in the context of Juniper devices, we refer to making standardised 
requests in order to execute show commands, or request various operations on the devices (such as reboot, software 
upgrade, clear DHCP leases, etc.), or configuration management oriented operations.

On Juniper, almost any CLI command has an RPC call equivalent. To see this, append `| display xml rpc` to your command. 
Examples:

```
apnic@router1> show version | display xml rpc
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos">
    <rpc>
        <get-software-information>
        </get-software-information>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

The RPC call for `show version` is `get-software-information` (under the `<rpc>` tag). In general, the RPC call for
_show_ commands are prefixed by `get-`, and sometimes suffixed by `-information`. Other examples:

```
apnic@router1>  show ospf neighbor | display xml rpc
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/23.2R1.14/junos">
    <rpc>
        <get-ospf-neighbor-information>
        </get-ospf-neighbor-information>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>

```

There are also commands that don't have an RPC call:

```
apnic@router1> show ntp associations | display xml rpc
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos">
    <message>
        xml rpc equivalent of this command is not available.
    </message>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

As it says under the `<message>` tag, the `show ntp associations` command doesn't have an RPC call associated with.

The RPC calls are important, as when executed, they provide a standardised response. From the command line, we can also 
visualise the RPC responses, by adding `| display xml` to the command:

```
apnic@router1> show ospf neighbor | display xml
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/23.2R1.14/junos">
    <ospf-neighbor-information xmlns="http://xml.juniper.net/junos/23.2R0/junos-routing">
        <ospf-neighbor>
            <neighbor-address>10.1.12.1</neighbor-address>
            <interface-name>ge-0/0/0.0</interface-name>
            <ospf-neighbor-state>Full</ospf-neighbor-state>
            <neighbor-id>10.4.1.2</neighbor-id>
            <neighbor-priority>128</neighbor-priority>
            <activity-timer>33</activity-timer>
        </ospf-neighbor>
        <ospf-neighbor>
            <neighbor-address>10.12.12.1</neighbor-address>
            <interface-name>ge-0/0/1.0</interface-name>
            <ospf-neighbor-state>Full</ospf-neighbor-state>
            <neighbor-id>10.4.2.2</neighbor-id>
            <neighbor-priority>1</neighbor-priority>
            <activity-timer>33</activity-timer>
        </ospf-neighbor>
    </ospf-neighbor-information>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

This command is safe for show commands, however remember that it displays the output _after_ the command has been 
executed:

```
apnic@router1> request system software delete jservices-mobile | display xml
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/23.2R1.14/junos">
    <output>
        Package jservices-mobile-x86-32-20230622.113721_builder_junos_232_r1 is deactivated
        Deleting jservices-mobile-x86-32-20230622.113721_builder_junos_232_r1 ...
    </output>
    <package-result>0</package-result>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

For example, the XML output above has been returned _after_ the `jservices-mobile` has been removed.

The data is returned in XML format, which is often difficult to work with. But the excellent `junos-eznc` library makes 
our life easier to parse it, extract the information, and present it into a more human-readable and programmable format.

When running under a Junos Proxy Minion, we only need to provide the RPC to the `junos.rpc` Salt function. For NAPALM 
Proxy Minions we have access to this functionaliy as well, using the `napalm.junos_rpc` function with the same input 
/ output as `junos.rpc`: using the `get-lldp-neighbors-information` RPC which corresponds to the `show lldp neighbors` 
CLI command, we can run
```bash
salt router1 napalm.junos_rpc get-ospf-neighbor-information
```


```bash
root@salt:~#  salt router1 napalm.junos_rpc get-ospf-neighbor-information

    ----------
    comment:
    out:
        ----------
        ospf-neighbor-information:
            ----------
            ospf-neighbor:
                |_
                  ----------
                  activity-timer:
                      31
                  interface-name:
                      ge-0/0/0.0
                  neighbor-address:
                      10.1.12.1
                  neighbor-id:
                      10.4.1.2
                  neighbor-priority:
                      128
                  ospf-neighbor-state:
                      Full
                |_
                  ----------
                  activity-timer:
                      31
                  interface-name:
                      ge-0/0/1.0
                  neighbor-address:
                      10.12.12.1
                  neighbor-id:
                      10.4.2.2
                  neighbor-priority:
                      1
                  ospf-neighbor-state:
                      Full
    result:
        True
root@salt:~#
```

Compare the XML document seen on the command line for `show ospf neighbour | display xml` with this one received via 
Salt. It may help if you run with `--out=raw` to see the exact Python structure.


The same pattern can be used for any other commands. For example `request`-type CLI calls:

```
apnic@router1> request system software delete jservices-mobile | display xml rpc
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos">
    <rpc>
        <request-package-delete>
                <package-name>jservices-mobile</package-name>
        </request-package-delete>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

In this case, the RPC is `request-package-delete`. But there's another element of importance here: underneath the 
`<request-package-delete>` tag, there's another one for `<package-name>`, with the value `jservices-mobile` as this is 
what we have requested.

From Salt, this request can be translated as:

```bash
salt router2 napalm.junos_rpc request-package-delete package-name=jservices-mobile
```

```bash
root@salt:~# salt router2 napalm.junos_rpc request-package-delete package-name=jservices-mobile
router2:
    ----------
    comment:
    out:
        ----------
        output:
            /packages/db/jservices-mobile-x86-32-20170601.185252_builder_junos_172_r1
    result:
        True
```

As previously, the RPC name comes as a positional argument, to the `napalm.junos_rpc` function, then the `package-name` 
as a keyword-value argument `package-name=jservices-mobile`.

This can be extended to any other RPC call, as complex as we need it to be. For example, if we want to see the 
statistics for the `ge-0/0/2` interface, we'd run the `show interfaces ge-0/0/2 statistics detail` show command. Let's 
check its RPC structure:

```
apnic@router1> show interfaces ge-0/0/2 statistics detail | display xml rpc
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos">
    <rpc>
        <get-interface-information>
                <statistics/>
                <detail/>
                <interface-name>ge-0/0/2</interface-name>
        </get-interface-information>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

Similar to the previous example, underneath the RPC tag `<get-interface-information>`, there are a few other tags: 
`<statistics/>`, `<detail/>`, and `<interface-name>ge-0/0/2</interface-name>`. While the latter, `interface-name` has 
a specific value, `ge-0/0/2`, as this is the interface we're interogating for, the other two tags don't have any value -
instead, they have an implicit logical value: True when set, False otherwise. For this reasoning, in Salt, we also 
provide _boolean_ values:

```
root@salt:~# salt router2 napalm.junos_rpc get-interface-information statistics=True detail=True interface-name=ge-0/0/2
router2:
    ----------
    comment:
    out:
        ----------
        interface-information:
            ----------
            physical-interface:
                ----------
                active-alarms:
                    ----------
                    interface-alarms:
                        ----------
                        alarm-not-present:
                active-defects:
                    ----------
                    interface-alarms:
                        ----------
                        alarm-not-present:
                admin-status:
                    up
                bpdu-error:
                    none
                current-physical-address:
                    52:54:00:6b:52:04

...
... snip ...
```

While the RPC calls are generally consistent across platforms, it may happen sometimes that some particular RPCs to 
differ from one version to another, and / or sometimes from one model to another.


## Part-2: Executing RPC calls on Arista switches

While on Juniper we had to identify a specific RPC call corresponding to a CLI command, the operations on Arista are 
simpler from this perspective, as we only need to know the CLI command. The communication channel is based on pyeAPI, an 
open source library maintained by Arista. eAPI is the proprietary API used by Arista, available on all the switches, and 
it is an HTTP-based API.

To check what eAPI would return, from the command line, we only need to prefix the command with `| json`:

```bash
spine1#show version
Arista vEOS
Hardware version:
Serial number:
System MAC address:  5254.00c0.882b

Software image version: 4.18.1F
Architecture:           i386
Internal build version: 4.18.1F-4591672.4181F
Internal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed

Uptime:                 1 day, 2 hours and 49 minutes
Total memory:           1893316 kB
Free memory:            690108 kB

spine1#show version | json 
{
    "modelName": "vEOS",
    "internalVersion": "4.18.1F-4591672.4181F",
    "systemMacAddress": "52:54:00:c0:88:2b",
    "serialNumber": "",
    "memTotal": 1893316,
    "bootupTimestamp": 1611589526.54,
    "memFree": 690700,
    "version": "4.18.1F",
    "architecture": "i386",
    "isIntlVersion": false,
    "internalBuildId": "6fcb426e-70a9-48b8-8958-54bb72ee28ed",
    "hardwareRevision": ""
}
```

Using `| json`, the command provides the same details, just in structured JSON format, which is then exposed by the eAPI 
and made available over HTTP requests.

Through Salt, we can use the `napalm.pyeapi_run_commands` to execute this command:
```bash
salt spine1 napalm.pyeapi_run_commands "show version"
```

```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version"
spine1:
    |_
      ----------
      architecture:
          i386
      bootupTimestamp:
          1611589526.53
      hardwareRevision:
      internalBuildId:
          6fcb426e-70a9-48b8-8958-54bb72ee28ed
      internalVersion:
          4.18.1F-4591672.4181F
      isIntlVersion:
          False
      memFree:
          690300
      memTotal:
          1893316
      modelName:
          vEOS
      serialNumber:
      systemMacAddress:
          52:54:00:c0:88:2b
      version:
          4.18.1F
```

By default, the function returns structured data, as provided by the eAPI. Alternatively, if we want the raw format, as 
text, for human readable output, we only need to pass in the `encoding=text` argument:
```bash
salt spine1 napalm.pyeapi_run_commands "show version" encoding=text
```


```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" encoding=text
spine1:
    - Arista vEOS
      Hardware version:
      Serial number:
      System MAC address:  5254.00c0.882b

      Software image version: 4.18.1F
      Architecture:           i386
      Internal build version: 4.18.1F-4591672.4181F
      Internal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed

      Uptime:                 1 day, 2 hours and 51 minutes
      Total memory:           1893316 kB
      Free memory:            690192 kB
```

Re-run both commands with `--out=raw`:
```bash
salt spine1 napalm.pyeapi_run_commands "show version" encoding=text --out=raw
```

```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" --out=raw
{'spine1': [{'modelName': 'vEOS', 'internalVersion': '4.18.1F-4591672.4181F', 'systemMacAddress': '52:54:00:c0:88:2b', 'serialNumber': '', 'memTotal': 1893316, 'bootupTimestamp': 1611589526.53, 'memFree': 689992, 'version': '4.18.1F', 'architecture': 'i386', 'isIntlVersion': False, 'internalBuildId': '6fcb426e-70a9-48b8-8958-54bb72ee28ed', 'hardwareRevision': ''}]}
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" encoding=text --out=raw
{'spine1': ['Arista vEOS\nHardware version:    \nSerial number:       \nSystem MAC address:  5254.00c0.882b\n\nSoftware image version: 4.18.1F\nArchitecture:           i386\nInternal build version: 4.18.1F-4591672.4181F\nInternal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed\n\nUptime:                 1 day, 2 hours and 52 minutes\nTotal memory:           1893316 kB\nFree memory:            690192 kB\n\n']}
```

Notice that in either case the output is a list, as `napalm.pyeapi_run_commands` is capable to run multiple commands at 
once:
```bash
salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log" encoding=text
```


```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log" encoding=text
spine1:
    - Arista vEOS
      Hardware version:
      Serial number:
      System MAC address:  5254.00c0.882b

      Software image version: 4.18.1F
      Architecture:           i386
      Internal build version: 4.18.1F-4591672.4181F
      Internal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed

      Uptime:                 1 day, 3 hours and 3 minutes
      Total memory:           1893316 kB
      Free memory:            672420 kB

    - Tue Jan 26 18:49:18 2021
      Timezone: UTC
      Clock source: local
    - Timestamp           Delta Begin Msg
      2021-01-25 15:46:38 000.000000 Normal boot stages started
      2021-01-25 15:46:38 000.057444 Normal boot stages complete
```

The result is in the order we have requested the commands to be executed. The format is text-mode, as this is what we 
requested using the `encoding=text` argument. Let's re-run without this argument:

```bash
salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log"
```

```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log"
spine1:
    The minion function caused an exception: Traceback (most recent call last):
      File "/usr/local/lib/python3.6/site-packages/salt/metaproxy/proxy.py", line 480, in thread_return
        opts, data, func, args, kwargs
      File "/usr/local/lib/python3.6/site-packages/salt/executors/direct_call.py", line 12, in execute
        return func(*args, **kwargs)
      File "/usr/local/lib/python3.6/site-packages/salt/utils/napalm.py", line 515, in func_wrapper
        ret = func(*args, **kwargs)
      File "/usr/local/lib/python3.6/site-packages/salt/modules/napalm_mod.py", line 1053, in pyeapi_run_commands
        return __salt__["pyeapi.run_commands"](*commands, **pyeapi_kwargs)
      File "/usr/local/lib/python3.6/site-packages/salt/modules/arista_pyeapi.py", line 390, in run_commands
        "run_commands", commands, encoding=encoding, send_enable=send_enable, **kwargs
      File "/usr/local/lib/python3.6/site-packages/salt/modules/arista_pyeapi.py", line 286, in call
        ret = getattr(conn, method)(*args, **kwargs)
      File "/usr/local/lib/python3.6/site-packages/pyeapi/client.py", line 743, in run_commands
        response = self._connection.execute(commands, encoding, **kwargs)
      File "/usr/local/lib/python3.6/site-packages/pyeapi/eapilib.py", line 550, in execute
        response = self.send(request)
      File "/usr/local/lib/python3.6/site-packages/pyeapi/eapilib.py", line 469, in send
        raise CommandError(code, msg, command_error=err, output=out)
    pyeapi.eapilib.CommandError: Error [1003]: CLI command 3 of 4 'show clock' failed: unconverted command
ERROR: Minions returned with non-zero exit code
```

The execution has failed! Salt tells us that `Minions returned with non-zero exit code`, while the exception trace says 
`CLI command 3 of 4 'show clock' failed: unconverted command`. To understand what happens, let's go back to our switch:

```bash
spine1#show clock
Tue Jan 26 19:00:56 2021
Timezone: UTC
Clock source: local
```

The syntax is correct, and returns as it should, however when adding `| json` it returns:

```
spine1#show clock | json
{
    "errors": [
        "This is an unconverted command"
    ]
}
```

Similar to Juniper, not all the commands are available via the API, unfortunately. A workaround to this might be the 
methodology presented in _Part-3_.

## Part-3: Executing commands over SSH via Netmiko

We met Netmiko in a previous lab, when we run under the Netmiko Proxy Minion. One of the functions available for Netmiko 
Minion was `netmiko.send_command`.

For NAPALM Minions, we can similarly executed commands via Netmiko, even though the underlying communication channel is 
established over a separate channel. This is mostly useful on devices that don't provide an API, such as Cisco IOS, but 
can also be useful on the rest, in some particular circumstances.

Let's take one of the leaf switches and run a very simple command:

```
leaf2#show clock
*19:15:03.350 UTC Tue Jan 26 2021
```

Execution this though Salt, using the `napalm.netmiko_commands` is as straightforward as:
```bash
salt leaf2 napalm.netmiko_commands "show clock"
```


```bash
root@salt:~# salt leaf2 napalm.netmiko_commands "show clock"
leaf2:
    - *02:28:33.662 UTC Mon Aug 12 2024
```

Similar to the `napalm.pyeapi_run_commands` function, `napalm.netmiko_commands` equally returns the output as a list, as 
it accepts one or more commands to be executed:

```bash
salt leaf2 napalm.netmiko_commands "show clock" "show users"
```


```bash
root@salt:~# salt leaf2 napalm.netmiko_commands "show clock" "show users"
leaf2:
    - *19:18:12.761 UTC Tue Jan 26 2021
    -     Line       User       Host(s)              Idle       Location
         1 vty 0     apnic      idle                 00:00:40 10.0.0.2
         2 vty 1     apnic      idle                 00:00:21 10.0.0.2
         3 vty 2     apnic      idle                 00:01:57 10.0.0.2
         4 vty 3     apnic      idle                 00:01:48 10.0.0.2
      *  5 vty 4     apnic      idle                 00:00:00 10.0.0.2

        Interface    User               Mode         Idle     Peer Address
```

There are no gotchas, as everything available on the command line, is also available though `napalm.netmiko_commands`, 
on any platform. For example, let's run the same commands on one of the core switches, which is running Cisco IOS-XR, 
and the two commands are equally available:

```bash
salt core1 napalm.netmiko_commands "show clock" "show users"
```

```bash
root@salt:~# salt core1 napalm.netmiko_commands "show clock" "show users"
core1:
    -
      Tue Jan 26 19:18:35.045 UTC
      19:18:35.055 UTC Tue Jan 26 2021
    -
      Tue Jan 26 19:18:35.645 UTC
         Line            User                 Service  Conns   Idle        Location
         vty0            apnic                ssh          0  00:00:12     10.0.0.2
      *  vty1            apnic                ssh          0  00:00:00     10.0.0.2
```

The output is however different, as `core1` is a different platform than `leaf2`.

But the greatest power of `napalm.netmiko_commands` is that it works on _any_ platform supported by Netmiko; as we've 
seen previously, on Junos there are commands that are not available via NETCONF:

```
apnic@router1> show system processes brief              
last pid:  7282;  load averages:  0.17,  0.47,  0.54  up 0+04:27:25    19:24:14
150 processes: 2 running, 147 sleeping, 1 waiting

Mem: 143M Active, 1445M Inact, 200M Wired, 204M Buf, 168M Free
Swap: 3072M Total, 3072M Free




apnic@router1> show system processes brief | display xml rpc 
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos">
    <message>
        xml rpc equivalent of this command is not available.
    </message>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

If we want that output (as text), however, we can use `napalm.netmiko_commands` instead:

```bash
salt router1 napalm.netmiko_commands "show system processes brief"
```

```bash
root@salt:~# salt router1 napalm.netmiko_commands "show system processes brief"
router1:
    -
      last pid:  7306;  load averages:  1.15,  0.69,  0.60  up 0+04:29:52    19:26:41
      153 processes: 3 running, 149 sleeping, 1 waiting

      Mem: 165M Active, 1446M Inact, 202M Wired, 204M Buf, 144M Free
      Swap: 3072M Total, 3072M Free

```

Not only that some commands aren't available via NETCONF, but sometimes Juniper restricts them (e.g., `request system 
license add` or `traceroute monitor` - and many others); in cases like this, `napalm.netmiko_commands` can be very 
handy to automate the execution of those commands!

In a similar way, we've seen above that `show clock` is not available over the eAPI on Arista switches, as the command
is not converted, and, again, `napalm.netmiko_commands` can be helpful in such cases:

```bash
salt spine1 napalm.netmiko_commands "show clock"
```

```bash
root@salt:~# salt spine1 napalm.netmiko_commands "show clock"
spine1:
    - Tue Jan 26 19:31:47 2021
      Timezone: UTC
      Clock source: local
```

While the downside of `napalm.netmiko_commands` is that it is executed on SSH connection (therefore slower) and it 
returns the output as text (however using TextFSM, as we've seen, we are able to extract the data we need), the 
possibilities as unlimited.

## Part-4: Uploading files using SCP

SCP (Secure Copy Protocol) is a means of securely transferring computer files between two hosts (either from local to 
remote, or between two remote hosts). SCP uses SSH for data transfer, and uses the same authentication mechanisms. For 
this reasoning, from this perspective, it makes it excellent to be used through Salt, as we already have all the 
authentication details available. As well as we are able to execute commands via Netmiko, over SSH, Salt provides us 
with the tooling for uploading or downloading files to & from the network devices using SCP.

For this, we can use the `napalm.scp_put` and `napalm.scp_get` functions, which already are aware of the authentication 
credentials. All we have to provide is the files locations and eventually additional options that largely depend on 
external factors, such as buffer size, socket timeout, or known hosts.

For example, on `router`, there's a file `/var/log/inventory`; we can check the contents by running:

```
apnic@router1> file show /var/log/inventory
Mar 28 18:45:54 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
Jan 27 12:55:25 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
Jan 27 12:55:38 vmx chassis, serial number VM601162CA2B
Jan 27 12:55:39 CB 0 - part number , serial number
Jan 27 12:55:39 CB 1 - part number , serial number
Jan 27 12:55:39 Routing Engine 0 - part number , serial number
Jan 27 12:56:14 FPC 0 CPU - part number RIOT, serial number BUILTIN
Jan 27 12:56:46 FPC 0 PIC 0 - part number , serial number
Jan 27 12:56:46 FPC 0 MIC 0 - part number , serial number
```

Using `napalm.scp_get` we can download this file:
```bash
salt router1 napalm.scp_get /var/log/inventory /tmp/router1 auto_add_policy=True
```

```bash
root@salt:~# salt router1 napalm.scp_get /var/log/inventory /tmp/router1 auto_add_policy=True
router1:
    None
```

Using this command, we copy the file `/var/log/inventory` from `router1`, to `/tmp/router`, which is the local path. The 
`auto_add_policy=True` option is necessary as `router1` is not added to the `know_hosts` SSH file. This option is 
disabled by default, for security reasons, as `napalm.scp_put` and `napalm.scp_get` allow you to upload / download files 
from any reachable host which is a potential security threat.

To verify that the file has been downloaded correctly, run:
```bash
salt router1 file.read /tmp/router1
```

```bash
root@salt:~# salt router1 file.read /tmp/router1
router1:
    Mar 28 18:45:54 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
    Jan 27 12:55:25 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
    Jan 27 12:55:38 vmx chassis, serial number VM601162CA2B
    Jan 27 12:55:39 CB 0 - part number , serial number
    Jan 27 12:55:39 CB 1 - part number , serial number
    Jan 27 12:55:39 Routing Engine 0 - part number , serial number
    Jan 27 12:56:14 FPC 0 CPU - part number RIOT, serial number BUILTIN
    Jan 27 12:56:46 FPC 0 PIC 0 - part number , serial number
    Jan 27 12:56:46 FPC 0 MIC 0 - part number , serial number
```

It works in the same way also with platforms that don't have UNIX filesystem, for example Cisco IOS. Let's type `dir 
flash:` to see what we have on the flash of `leaf1` switch (on the command line, or through Salt):

```bash
salt leaf1 net.cli 'dir flash:'
```

```bash
root@salt:~# salt leaf1 net.cli 'dir flash:'
leaf1:
    ----------
    comment:
    out:
        ----------
        dir flash::
            Directory of bootflash:/

               11  drwx            16384  Mar 24 2015 00:52:21 +00:00  lost+found
            324481  drwx             4096  Mar 24 2015 00:53:30 +00:00  .super.iso.dir
               12  -rw-               45  Jan 27 2021 12:56:22 +00:00  .CsrLxc_LastInstall
               13  -rw-               83  Mar 24 2015 00:56:54 +00:00  virtual-instance.conf
            665185  drwx             4096  Mar 24 2015 00:55:49 +00:00  core
               15  -rw-        161894400  Mar 24 2015 00:53:28 +00:00  iosxe-remote-mgmt.03.15.00.S.155-2.S-std.ova
            64900  -rw-        282431424  Mar 24 2015 00:54:35 +00:00  csr1000v-mono-universalk9.03.15.00.S.155-2.S-std.SPA.pkg
            64898  -rw-             4874  Mar 24 2015 00:54:35 +00:00  csr1000v-packages-universalk9.03.15.00.S.155-2.S-std.conf
            64899  -rw-             5662  Mar 24 2015 00:54:35 +00:00  packages.conf
            486721  drwx             4096  Mar 24 2015 00:55:49 +00:00  .prst_sync
            389377  drwx             4096  Mar 24 2015 00:55:53 +00:00  .rollback_timer
               16  -rw-                0  Mar 24 2015 00:55:55 +00:00  tracelogs.137
            32449  drwx             4096  Jan 27 2021 13:01:44 +00:00  tracelogs
            454273  drwx             4096  Mar 24 2015 00:56:03 +00:00  .installer
               17  -rw-                0  Jan 27 2021 12:56:48 +00:00  cvac.log
               18  -rw-             1552  Jan 27 2021 12:57:03 +00:00  csrlxc-cfg.log
            600289  drwx             4096  Mar 24 2015 00:57:02 +00:00  onep
               19  -rw-               16   Dec 7 2020 19:42:53 +00:00  iosxe_config.txt.md5
               20  -rw-               73  Jan 27 2021 13:00:34 +00:00  merge_config.txt
            373153  drwx             4096   Dec 7 2020 19:42:20 +00:00  .ssh
               21  -rw-             1890  Jan 27 2021 13:01:17 +00:00  rollback_config.txt
               22  -rw-             1890  Jan 27 2021 13:00:46 +00:00  archive-Jan-27-13-00-46.294-0
               23  -rw-             2062  Jan 27 2021 13:01:08 +00:00  candidate_config.txt
               24  -rw-             1890  Jan 27 2021 13:01:19 +00:00  archive-Jan-27-13-01-19.527-1
               25  -rw-             2102  Jan 27 2021 13:01:21 +00:00  archive-Jan-27-13-01-21.495-2

            7835619328 bytes total (6547709952 bytes free)
    result:
        True
```

One of the files under `flash:` is `packages.conf`, with this contents:
```bash
salt leaf1 net.cli 'more flash:packages.conf'
```

```bash
root@salt:~# salt leaf1 net.cli 'more flash:packages.conf'
leaf1:
    ----------
    comment:
    out:
        ----------
         more flash:packages.conf:
            #! /usr/binos/bin/packages_conf.sh

            sha1sum: 46482bb61607b01d831ce9eb1298bef9ea137020
            boot  rp 0 0   rp_boot       csr1000v-rpboot.16.12.04.SPA.pkg

            iso   rp 0 0   rp_base       csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 1   rp_base       csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 0   rp_daemons    csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 1   rp_daemons    csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 0   rp_iosd       csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 1   rp_iosd       csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 0   rp_security   csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 1   rp_security   csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   fp 0 0   fp            csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   fp 0 1   fp            csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 0   rp_webui      csr1000v-mono-universalk9.16.12.04.SPA.pkg
            iso   rp 0 1   rp_webui      csr1000v-mono-universalk9.16.12.04.SPA.pkg
    result:
        True
```

With the following commands we can download it, then check the contents:

```bash
salt leaf1 napalm.scp_get flash:packages.conf /tmp/leaf1 auto_add_policy=True
salt leaf1 file.read /tmp/leaf1

```

```bash
root@salt:~# salt leaf1 napalm.scp_get flash:packages.conf /tmp/leaf1 auto_add_policy=True
leaf1:
    None
root@salt:~# salt leaf1 file.read /tmp/leaf1
leaf1:
    #! /usr/binos/bin/packages_conf.sh

    sha1sum: 46482bb61607b01d831ce9eb1298bef9ea137020
    boot  rp 0 0   rp_boot       csr1000v-rpboot.16.12.04.SPA.pkg

    iso   rp 0 0   rp_base       csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 1   rp_base       csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 0   rp_daemons    csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 1   rp_daemons    csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 0   rp_iosd       csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 1   rp_iosd       csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 0   rp_security   csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 1   rp_security   csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   fp 0 0   fp            csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   fp 0 1   fp            csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 0   rp_webui      csr1000v-mono-universalk9.16.12.04.SPA.pkg
    iso   rp 0 1   rp_webui      csr1000v-mono-universalk9.16.12.04.SPA.pkg

```

`napalm.scp_put` works in the exact same way, just that the files are transferred in the opposite direction (from Salt 
to the network device). We've downloaded earlier the file `/tmp/router1`, on the `router1` Minion. We can upload it 
back, to the `/var/tmp` directory on the `router1`:

```bash
salt router1 napalm.scp_put /tmp/router1 /var/tmp/inventory auto_add_policy=True
```

```bash
root@salt:~# salt router1 napalm.scp_put /tmp/router1 /var/tmp/inventory auto_add_policy=True
router1:
    None
```

And using the `net.cli` Salt function we can verify that it has been uploaded indeed:

```bash
 salt router1 net.cli 'file list /var/tmp/inventory'
 salt router1 net.cli 'file show /var/tmp/inventory'
 ```

```bash
root@salt:~# salt router1 net.cli 'file list /var/tmp/inventory'
router1:
    ----------
    comment:
    out:
        ----------
        file list /var/tmp/inventory:
            /var/tmp/inventory
    result:
        True
root@salt:~# salt router1 net.cli 'file show /var/tmp/inventory'
router1:
    ----------
    comment:
    out:
        ----------
        file show /var/tmp/inventory:
            Mar 28 18:45:54 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
            Jan 27 12:55:25 CHASSISD release 17.2R1.13 built by builder on 2017-06-01 22:55:39 UTC
            Jan 27 12:55:38 vmx chassis, serial number VM601162CA2B
            Jan 27 12:55:39 CB 0 - part number , serial number
            Jan 27 12:55:39 CB 1 - part number , serial number
            Jan 27 12:55:39 Routing Engine 0 - part number , serial number
            Jan 27 12:56:14 FPC 0 CPU - part number RIOT, serial number BUILTIN
            Jan 27 12:56:46 FPC 0 PIC 0 - part number , serial number
            Jan 27 12:56:46 FPC 0 MIC 0 - part number , serial number
    result:
        True
```

One of the greatest benefits of `napalm.scp_put` is that the source file can be referenced using the absolute path, or 
using the usual URIs Salt can work with: `salt://`, `http(s)://`, `s3://`, `swift://`, etc. For example, let's upload 
one of the files we've had in the previous examples, `salt://static/junos`:

```bash
salt router1 napalm.scp_put salt://static/junos /var/home/admin/ auto_add_policy=True
```

```bash
root@salt:~# salt router1 napalm.scp_put salt://static/junos /var/home/admin/ auto_add_policy=True
router1:
    None
```

The file has been uploaded to our home directory (for the `admin` user):

```bash
salt router1 net.cli 'file list /var/home/admin'
salt router1 net.cli 'file show /var/home/apnic/junos'
```

```bash
root@salt:~# salt router1 net.cli 'file list /var/home/admin'
router1:
    ----------
    comment:
    out:
        ----------
        file list /var/home/apnic:
            /var/home/apnic:
            junos
    result:
        True

root@salt:~# salt router1 net.cli 'file show /var/home/apnic/junos'
router1:
    ----------
    comment:
    out:
        ----------
        file show /var/home/apnic/junos:
            set system ntp server 10.0.0.1
    result:
        True
```

While these two functions are simple enough to operate and understand, they can be extremely handy in various automation 
contexts, e.g., upload system license file, load configuration, or download bulk logs, etc.

--
**End of Lab**

---
