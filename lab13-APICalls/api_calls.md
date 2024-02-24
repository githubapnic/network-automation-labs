![](images/apnic_logo.png)
# LAB: Invoking low-level API functions

For this lab, we will have again all the Proxy Minions running, without any other external services. All the Minions are running using NAPALM for the connection.

Low-level API functions allow you to interact directly with the network device, and present you the data exactly in the way the device returns it. We've seen previously functions such as `net.arp`, `net.lldp`, and others; under the hood, these functions execute one or more low-level API calls, and eventually some data processing, in order to provide the data in a vendor-agnostic format, easy to consume. But these low-level API calls differ from one platform to another. In the next sections we will look into what Salt functions you can use per individual platform. In fact, we've already seen some of those in the previous lab, but now, we'll look into them closely.

First, lets make sure that we have all of our router configs loaded.

```bash
salt \* lab.restore -t 120
```

### Logging into the router
There may be an issue with a conflict with the fingerprint used for previous labs. 

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
  ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "router1"
Host key for router1 has changed and you have requested strict checking.
Host key verification failed.
</pre>

The easiest fix is to delete the old key from the known_hosts file.

<pre>
ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "router1"
</pre>

Otherwise, to bypass this check you can try this command:

```bash
ssh -o "StrictHostKeyChecking no" apnic@router1
```

**password** = APNIC2021

## Part-1: Executing RPC calls on Juniper devices

RPC (Remote Procedure Call) is when a computer program causes a procedure (subroutine, or instruction) on another, remote machine. When talking about RPC calls in the context of Juniper devices, we refer to making standardised requests in order to execute show commands, or request various operations on the devices (such as reboot, software upgrade, clear DHCP leases, etc.), or configuration management oriented operations.

On Juniper, almost any CLI command has an RPC call equivalent. To see this, append `| display xml rpc` to your command. 

Examples:

```
show version | display xml rpc
```

<pre>
apnic@router1&gt; show version | display xml rpc
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;rpc&gt;
        &lt;get-software-information&gt;
        &lt;/get-software-information&gt;
    &lt;/rpc&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

The RPC call for `show version` is `get-software-information` (under the `<rpc>` tag). In general, the RPC call for _show_ commands are prefixed by `get-`, and sometimes suffixed by `-information`. Other examples:

```
show lldp neighbors | display xml rpc
```

<pre>
apnic@router1&gt; show lldp neighbors | display xml rpc
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;rpc&gt;
        &lt;get-lldp-neighbors-information&gt;
        &lt;/get-lldp-neighbors-information&gt;
    &lt;/rpc&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

```
show chassis alarms | display xml rpc
```

<pre>
apnic@router1&gt; show chassis alarms | display xml rpc 
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;rpc&gt;
        &lt;get-alarm-information&gt;
        &lt;/get-alarm-information&gt;
    &lt;/rpc&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

There are also commands that don't have an RPC call:

```
show ntp associations | display xml rpc
```

<pre>
apnic@router1&gt; show ntp associations | display xml rpc
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;message&gt;
        xml rpc equivalent of this command is not available.
    &lt;/message&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

As it says under the `<message>` tag, the `show ntp associations` command doesn't have an RPC call associated with.

The RPC calls are important, as when executed, they provide a standardised response. From the command line, we can also visualise the RPC responses, by adding `| display xml` to the command:

```
show lldp neighbors | display xml
```

<pre>
apnic@router1&gt; show lldp neighbors | display xml
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;lldp-neighbors-information junos:style="brief"&gt;
        &lt;lldp-neighbor-information&gt;
            &lt;lldp-local-port-id&gt;ge-0/0/0&lt;/lldp-local-port-id&gt;
            &lt;lldp-local-parent-interface-name&gt;-&lt;/lldp-local-parent-interface-name&gt;
            &lt;lldp-remote-chassis-id-subtype&gt;Mac address&lt;/lldp-remote-chassis-id-subtype&gt;
            &lt;lldp-remote-chassis-id&gt;00:05:86:44:79:c0&lt;/lldp-remote-chassis-id&gt;
            &lt;lldp-remote-port-id-subtype&gt;Locally assigned&lt;/lldp-remote-port-id-subtype&gt;
            &lt;lldp-remote-port-id&gt;518&lt;/lldp-remote-port-id&gt;
            &lt;lldp-remote-system-name&gt;router2&lt;/lldp-remote-system-name&gt;
        &lt;/lldp-neighbor-information&gt;
        &lt;lldp-neighbor-information&gt;
            &lt;lldp-local-port-id&gt;ge-0/0/2&lt;/lldp-local-port-id&gt;
            &lt;lldp-local-parent-interface-name&gt;-&lt;/lldp-local-parent-interface-name&gt;
            &lt;lldp-remote-chassis-id-subtype&gt;Mac address&lt;/lldp-remote-chassis-id-subtype&gt;
            &lt;lldp-remote-chassis-id&gt;02:07:e0:bd:0c:06&lt;/lldp-remote-chassis-id&gt;
            &lt;lldp-remote-port-id-subtype&gt;Interface name&lt;/lldp-remote-port-id-subtype&gt;
            &lt;lldp-remote-port-id&gt;Gi0/0/0/3&lt;/lldp-remote-port-id&gt;
            &lt;lldp-remote-system-name&gt;core2&lt;/lldp-remote-system-name&gt;
        &lt;/lldp-neighbor-information&gt;
        &lt;lldp-neighbor-information&gt;
            &lt;lldp-local-port-id&gt;ge-0/0/1&lt;/lldp-local-port-id&gt;
            &lt;lldp-local-parent-interface-name&gt;-&lt;/lldp-local-parent-interface-name&gt;
            &lt;lldp-remote-chassis-id-subtype&gt;Mac address&lt;/lldp-remote-chassis-id-subtype&gt;
            &lt;lldp-remote-chassis-id&gt;02:28:b5:5a:a4:06&lt;/lldp-remote-chassis-id&gt;
            &lt;lldp-remote-port-id-subtype&gt;Interface name&lt;/lldp-remote-port-id-subtype&gt;
            &lt;lldp-remote-port-id&gt;Gi0/0/0/2&lt;/lldp-remote-port-id&gt;
            &lt;lldp-remote-system-name&gt;core1&lt;/lldp-remote-system-name&gt;
        &lt;/lldp-neighbor-information&gt;
    &lt;/lldp-neighbors-information&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

This command is safe for show commands, however remember that it displays the output _after_ the command has been executed:

```
request system software delete jservices-mobile | display xml
```

<pre>
apnic@router1&gt; request system software delete jservices-mobile | display xml
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;output&gt;
        /packages/db/jservices-mobile-x86-32-20170601.185252_builder_junos_172_r1
    &lt;/output&gt;
    &lt;package-result&gt;0&lt;/package-result&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

For example, the XML output above has been returned _after_ the `jservices-mobile` has been removed.

The data is returned in XML format, which is often difficult to work with. But the excellent `junos-eznc` library makes our life easier to parse it, extract the information, and present it into a more human-readable and programmable format.

When running under a Junos Proxy Minion, we only need to provide the RPC to the `junos.rpc` Salt function. For NAPALM Proxy Minions we have access to this functionaliy as well, using the `napalm.junos_rpc` function with the same input/output as `junos.rpc`: using the `get-lldp-neighbors-information` RPC which corresponds to the `show lldp neighbors` CLI command, we can run

Open a new termrinal window, to run the salt commands.

```bash
salt router1 napalm.junos_rpc get-lldp-neighbors-information
```

<pre>
root@salt:~# salt router1 napalm.junos_rpc get-lldp-neighbors-information
router1:
    ----------
    comment:
    out:
        ----------
        lldp-neighbors-information:
            ----------
            lldp-neighbor-information:
                |_
                  ----------
                  lldp-local-parent-interface-name:
                      -
                  lldp-local-port-id:
                      ge-0/0/0
                  lldp-remote-chassis-id:
                      00:05:86:44:79:c0
                  lldp-remote-chassis-id-subtype:
                      Mac address
                  lldp-remote-port-id:
                      518
                  lldp-remote-port-id-subtype:
                      Locally assigned
                  lldp-remote-system-name:
                      router2
                |_
                  ----------
                  lldp-local-parent-interface-name:
                      -
                  lldp-local-port-id:
                      ge-0/0/2
                  lldp-remote-chassis-id:
                      02:07:e0:bd:0c:06
                  lldp-remote-chassis-id-subtype:
                      Mac address
                  lldp-remote-port-id:
                      Gi0/0/0/3
                  lldp-remote-port-id-subtype:
                      Interface name
                  lldp-remote-system-name:
                      core2
                |_
                  ----------
                  lldp-local-parent-interface-name:
                      -
                  lldp-local-port-id:
                      ge-0/0/1
                  lldp-remote-chassis-id:
                      02:28:b5:5a:a4:06
                  lldp-remote-chassis-id-subtype:
                      Mac address
                  lldp-remote-port-id:
                      Gi0/0/0/2
                  lldp-remote-port-id-subtype:
                      Interface name
                  lldp-remote-system-name:
                      core1
    result:
        True
</pre>

Compare the XML document seen on the command line for `show lldp neighors | display xml` with this one received via Salt. It may help if you run with `--out=raw` to see the exact Python structure.

Notice that in the XML document on the CLI, we had the `lldp-neighbor-information` tag multiple times. In the Python document, as it is repetitive it becomes a list, each element of the list being a Python dictionary (i.e., a hash mapping) with the `lldp-local-port-id`, `lldp-remote-chassis-id`, etc. elements.

The same pattern can be used for any other commands. For example `request`-type CLI calls:

Return to the terminal window that is connected to router1.

```
request system software delete jservices-mobile | display xml rpc
```

<pre>
apnic@router1&gt; request system software delete jservices-mobile | display xml rpc
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;rpc&gt;
        &lt;request-package-delete&gt;
                &lt;package-name&gt;jservices-mobile&lt;/package-name&gt;
        &lt;/request-package-delete&gt;
    &lt;/rpc&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

In this case, the RPC is `request-package-delete`. But there's another element of importance here: underneath the `<request-package-delete>` tag, there's another one for `<package-name>`, with the value `jservices-mobile` as this is what we have requested.

Return to the terminal window that is used for the salt commands. From Salt, this request can be translated as:

```bash
salt router2 napalm.junos_rpc request-package-delete package-name=jservices-mobile
```

<pre>
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
</pre>

As previously, the RPC name comes as a positional argument, to the `napalm.junos_rpc` function, then the `package-name` as a keyword-value argument `package-name=jservices-mobile`.

This can be extended to any other RPC call, as complex as we need it to be. For example, if we want to see the statistics for the `ge-0/0/2` interface, we'd run the `show interfaces ge-0/0/2 statistics detail` show command. Let's check its RPC structure:

Return to the terminal window that is connected to router1.

```
show interfaces ge-0/0/2 statistics detail | display xml rpc
```

<pre>
apnic@router1&gt; show interfaces ge-0/0/2 statistics detail | display xml rpc
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;rpc&gt;
        &lt;get-interface-information&gt;
                &lt;statistics/&gt;
                &lt;detail/&gt;
                &lt;interface-name&gt;ge-0/0/2&lt;/interface-name&gt;
        &lt;/get-interface-information&gt;
    &lt;/rpc&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

Similar to the previous example, underneath the RPC tag `<get-interface-information>`, there are a few other tags: 
`<statistics/>`, `<detail/>`, and `<interface-name>ge-0/0/2</interface-name>`. While the latter, `interface-name` has a specific value, `ge-0/0/2`, as this is the interface we're interogating for, the other two tags don't have any value - instead, they have an implicit logical value: True when set, False otherwise. For this reasoning, in Salt, we also provide _boolean_ values:

Return to the terminal window that is used for the salt commands. From Salt, this request can be translated as:

```
salt router2 napalm.junos_rpc get-interface-information statistics=True detail=True interface-name=ge-0/0/2
```

<pre>
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
</pre>

While the RPC calls are generally consistent across platforms, it may happen sometimes that some particular RPCs to differ from one version to another, and / or sometimes from one model to another.

Return to the terminal window that is connected to router1, and exit the ssh session.

```
exit
```

## Part-2: Executing RPC calls on Arista switches

### Logging into the switch
There may be an issue with a conflict with the fingerprint used for previous labs. 

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
  ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "spine1"
Host key for router1 has changed and you have requested strict checking.
Host key verification failed.
</pre>

The easiest fix is to delete the old key from the known_hosts file.

<pre>
ssh-keygen -f "/home/apnic/.ssh/known_hosts" -R "spine1"
</pre>

Otherwise, to bypass this check you can try this command:

```bash
ssh -o "StrictHostKeyChecking no" apnic@spine1
```

**password** = APNIC2021

While on Juniper we had to identify a specific RPC call corresponding to a CLI command, the operations on Arista are simpler from this perspective, as we only need to know the CLI command. The communication channel is based on pyeAPI, an open source library maintained by Arista. eAPI is the proprietary API used by Arista, available on all the switches, and it is an HTTP-based API.

To check what eAPI would return, from the command line, we only need to prefix the command with `| json`:

```bash
show version
```

<pre>
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
</pre>

```
show version | json 
```

<pre>
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
</pre>

Using `| json`, the command provides the same details, just in structured JSON format, which is then exposed by the eAPI and made available over HTTP requests.

Return to the terminal window that is used for the salt commands. Through Salt, we can use the `napalm.pyeapi_run_commands` to execute this command:

```bash
salt spine1 napalm.pyeapi_run_commands "show version"
```

<pre>
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
</pre>

By default, the function returns structured data, as provided by the eAPI. Alternatively, if we want the raw format, as text, for human readable output, we only need to pass in the `encoding=text` argument:

```bash
salt spine1 napalm.pyeapi_run_commands "show version" encoding=text
```

<pre>
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
</pre>

Re-run both commands with `--out=raw`:

```bash
salt spine1 napalm.pyeapi_run_commands "show version" --out=raw
```

<pre>
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" --out=raw
{'spine1': [{'modelName': 'vEOS', 'internalVersion': '4.18.1F-4591672.4181F', 'systemMacAddress': '52:54:00:c0:88:2b', 'serialNumber': '', 'memTotal': 1893316, 'bootupTimestamp': 1611589526.53, 'memFree': 689992, 'version': '4.18.1F', 'architecture': 'i386', 'isIntlVersion': False, 'internalBuildId': '6fcb426e-70a9-48b8-8958-54bb72ee28ed', 'hardwareRevision': ''}]}
</pre>

```bash
salt spine1 napalm.pyeapi_run_commands "show version" encoding=text --out=raw
```

<pre>
root@salt:~# salt spine1 napalm.pyeapi_run_commands "show version" encoding=text --out=raw
{'spine1': ['Arista vEOS\nHardware version:    \nSerial number:       \nSystem MAC address:  5254.00c0.882b\n\nSoftware image version: 4.18.1F\nArchitecture:           i386\nInternal build version: 4.18.1F-4591672.4181F\nInternal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed\n\nUptime:                 1 day, 2 hours and 52 minutes\nTotal memory:           1893316 kB\nFree memory:            690192 kB\n\n']}
</pre>

Notice that in either case the output is a list, as `napalm.pyeapi_run_commands` is capable to run multiple commands at once:

```bash
salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log" encoding=text
```

<pre>
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
</pre>

The result is in the order we have requested the commands to be executed. The format is text-mode, as this is what we requested using the `encoding=text` argument. Let's re-run without this argument:

```bash
salt spine1 napalm.pyeapi_run_commands "show version" "show clock" "show boot stages log"
```

<pre>
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
</pre>

The execution has failed! Salt tells us that `Minions returned with non-zero exit code`, while the exception trace says `CLI command 3 of 4 'show clock' failed: unconverted command`. To understand what happens, let's go back to our switch:

```bash
show clock
```

<pre>
spine1#show clock
Tue Jan 26 19:00:56 2021
Timezone: UTC
Clock source: local
</pre>

The syntax is correct, and returns as it should, however when adding `| json` it returns:

```
spine1#show clock | json
```

<pre>
spine1#show clock | json
{
    "errors": [
        "This is an unconverted command"
    ]
}
</pre>

Similar to Juniper, not all the commands are available via the API, unfortunately. A workaround to this might be the methodology presented in _Part-3_.

Exit the ssh session for spine1.

```
exit
```

## Part-3: Executing commands over SSH via Netmiko

Log into the leaf2 switch

```bash
ssh -o "StrictHostKeyChecking no" apnic@leaf2
```

**password** = APNIC2021

We met Netmiko in a previous lab, when we run under the Netmiko Proxy Minion. One of the functions available for Netmiko Minion was `netmiko.send_command`.

For NAPALM Minions, we can similarly executed commands via Netmiko, even though the underlying communication channel is established over a separate channel. This is mostly useful on devices that don't provide an API, such as Cisco IOS, but can also be useful on the rest, in some particular circumstances.

Let's take one of the leaf switches and run a very simple command:

```
show clock
```

<pre>
leaf2#show clock
*19:15:03.350 UTC Tue Jan 26 2021
</pre>

Return to the terminal window that is used for the salt commands. Execute this through Salt, using the `napalm.netmiko_commands` is as straightforward as:

```bash
salt leaf2 napalm.netmiko_commands "show clock"
```

<pre>
root@salt:~# salt leaf2 napalm.netmiko_commands "show clock"
leaf2:
    - *19:16:25.385 UTC Tue Jan 26 2021
</pre>

Similar to the `napalm.pyeapi_run_commands` function, `napalm.netmiko_commands` equally returns the output as a list, as it accepts one or more commands to be executed:

```bash
salt leaf2 napalm.netmiko_commands "show clock" "show users"
```

<pre>
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
</pre>

There are no gotchas, as everything available on the command line, is also available though `napalm.netmiko_commands`, on any platform. For example, let's run the same commands on one of the core switches, which is running Cisco IOS-XR, and the two commands are equally available:

```bash
salt core1 napalm.netmiko_commands "show clock" "show users"
```

<pre>
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
</pre>

The output is however different, as `core1` is a different platform than `leaf2`.

But the greatest power of `napalm.netmiko_commands` is that it works on _any_ platform supported by Netmiko; as we've seen previously, on Junos there are commands that are not available via NETCONF:

Open a new terminal window and log into router1

```bash
ssh -o "StrictHostKeyChecking no" apnic@router1
```

**password** = APNIC2021

```
show system processes brief
```

<pre>
apnic@router1> show system processes brief   
last pid:  7282;  load averages:  0.17,  0.47,  0.54  up 0+04:27:25    19:24:14
150 processes: 2 running, 147 sleeping, 1 waiting

Mem: 143M Active, 1445M Inact, 200M Wired, 204M Buf, 168M Free
Swap: 3072M Total, 3072M Free
</pre>

```
show system processes brief | display xml rpc
```

<pre>
apnic@router1&gt; show system processes brief | display xml rpc 
&lt;rpc-reply xmlns:junos="http://xml.juniper.net/junos/17.2R1/junos"&gt;
    &lt;message&gt;
        xml rpc equivalent of this command is not available.
    &lt;/message&gt;
    &lt;cli&gt;
        &lt;banner&gt;&lt;/banner&gt;
    &lt;/cli&gt;
&lt;/rpc-reply&gt;
</pre>

If we want that output (as text), however, we can use `napalm.netmiko_commands` instead:

Return to the terminal window that is used for the salt commands. 

```bash
salt router1 napalm.netmiko_commands "show system processes brief"
```

<pre>
root@salt:~# salt router1 napalm.netmiko_commands "show system processes brief"
router1:
    -
      last pid:  7306;  load averages:  1.15,  0.69,  0.60  up 0+04:29:52    19:26:41
      153 processes: 3 running, 149 sleeping, 1 waiting

      Mem: 165M Active, 1446M Inact, 202M Wired, 204M Buf, 144M Free
      Swap: 3072M Total, 3072M Free
</pre>

Not only that some commands aren't available via NETCONF, but sometimes Juniper restricts them (e.g., `request system license add` or `traceroute monitor` - and many others); in cases like this, `napalm.netmiko_commands` can be very handy to automate the execution of those commands!

In a similar way, we've seen above that `show clock` is not available over the eAPI on Arista switches, as the command is not converted, and, again, `napalm.netmiko_commands` can be helpful in such cases:

```bash
salt spine1 napalm.netmiko_commands "show clock"
```

<pre>
root@salt:~# salt spine1 napalm.netmiko_commands "show clock"
spine1:
    - Tue Jan 26 19:31:47 2021
      Timezone: UTC
      Clock source: local
</pre>

While the downside of `napalm.netmiko_commands` is that it is executed on SSH connection (therefore slower) and it returns the output as text (however using TextFSM, as we've seen, we are able to extract the data we need), the possibilities as unlimited.

## Part-4: Uploading files using SCP

SCP (Secure Copy Protocol) is a means of securely transferring computer files between two hosts (either from local to remote, or between two remote hosts). SCP uses SSH for data transfer, and uses the same authentication mechanisms. For this reasoning, from this perspective, it makes it excellent to be used through Salt, as we already have all the authentication details available. As well as we are able to execute commands via Netmiko, over SSH, Salt provides us with the tooling for uploading or downloading files to & from the network devices using SCP.

For this, we can use the `napalm.scp_put` and `napalm.scp_get` functions, which already are aware of the authentication credentials. All we have to provide is the files locations and eventually additional options that largely depend on external factors, such as buffer size, socket timeout, or known hosts.

Return to the terminal window that is connected to router1. For example, on `router`, there's a file `/var/log/inventory`; we can check the contents by running:

```
file show /var/log/inventory
```

<pre>
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
</pre>

Return to the terminal window that is used for the salt commands. Using `napalm.scp_get` we can download this file:

```bash
salt router1 napalm.scp_get /var/log/inventory /tmp/router1 auto_add_policy=True
```

<pre>
root@salt:~# salt router1 napalm.scp_get /var/log/inventory /tmp/router1 auto_add_policy=True
router1:
    None
</pre>

Using this command, we copy the file `/var/log/inventory` from `router1`, to `/tmp/router`, which is the local path. The `auto_add_policy=True` option is necessary as `router1` is not added to the `know_hosts` SSH file. This option is disabled by default, for security reasons, as `napalm.scp_put` and `napalm.scp_get` allow you to upload / download files from any reachable host which is a potential security threat.

To verify that the file has been downloaded correctly, run:

```bash
salt router1 file.read /tmp/router1
```

<pre>
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
</pre>

It works in the same way also with platforms that don't have UNIX filesystem, for example Cisco IOS. Let's type `dir flash:` to see what we have on the flash of `leaf1` switch (on the command line, or through Salt):

```bash
salt leaf1 net.cli 'dir flash:'
```

<pre>
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
</pre>

One of the files under `flash:` is `virtual-instance.conf`, with this contents:

```bash
salt leaf1 net.cli 'more flash:virtual-instance.conf'
```

<pre>
root@salt:~# salt leaf1 net.cli 'more flash:virtual-instance.conf'
leaf1:
    ----------
    comment:
    out:
        ----------
        more flash:virtual-instance.conf:
            rp 0 0 csr_mgmt /bootflash/iosxe-remote-mgmt.03.15.00.S.155-2.S-std.ova /bootflash
    result:
        True
</pre>

With the following commands we can download it, then check the contents:

```bash
salt leaf1 napalm.scp_get flash:virtual-instance.conf /tmp/leaf1 auto_add_policy=True
```

<pre>
root@salt:~# salt leaf1 napalm.scp_get flash:virtual-instance.conf /tmp/leaf1 auto_add_policy=True
leaf1:
    None
</pre>

```bash
salt leaf1 file.read /tmp/leaf1
```

<pre>
root@salt:~# salt leaf1 file.read /tmp/leaf1
leaf1:
    rp 0 0 csr_mgmt /bootflash/iosxe-remote-mgmt.03.15.00.S.155-2.S-std.ova /bootflash
</pre>

`napalm.scp_put` works in the exact same way, just that the files are transferred in the opposite direction (from Salt to the network device). We've downloaded earlier the file `/tmp/router1`, on the `router1` Minion. We can upload it back, to the `/var/tmp` directory on the `router1`:

```bash
salt router1 napalm.scp_put /tmp/router1 /var/tmp/inventory auto_add_policy=True
```

<pre>
root@salt:~# salt router1 napalm.scp_put /tmp/router1 /var/tmp/inventory auto_add_policy=True
router1:
    None
</pre>

And using the `net.cli` Salt function we can verify that it has been uploaded indeed:

```bash
salt router1 net.cli 'file list /var/tmp/inventory'
```

<pre>
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
</pre>

```bash
salt router1 net.cli 'file show /var/tmp/inventory'
```

<pre>
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
</pre>

One of the greatest benefits of `napalm.scp_put` is that the source file can be referenced using the absolute path, or using the usual URIs Salt can work with: `salt://`, `http(s)://`, `s3://`, `swift://`, etc. For example, let's upload one of the files we've had in the previous examples, `salt://static/junos`:

```bash
salt router1 napalm.scp_put salt://static/junos /var/home/apnic/ auto_add_policy=True
```

<pre>
root@salt:~# salt router1 napalm.scp_put salt://static/junos /var/home/apnic/ auto_add_policy=True
router1:
    None
</pre>

The file has been uploaded to our home directory (for the `apnic` user):

```bash
salt router1 net.cli 'file list /var/home/apnic'
```

<pre>
root@salt:~# salt router1 net.cli 'file list /var/home/apnic'
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
</pre>

```bash
salt router1 net.cli 'file show /var/home/apnic/junos'
```

<pre>
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
</pre>

While these two functions are simple enough to operate and understand, they can be extremely handy in various automation contexts, e.g., upload system license file, load configuration, or download bulk logs, etc.

--
**End of Lab**

---
