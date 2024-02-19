![](images/apnic_logo.png)
# LAB: Configuring Proxy Minions

We will need to install some pre-requesit packages.  The latest version of salt has a one-dir install with the correct version of python.
The following will install with one command

- `NAPALM`
- `Netmiko`
- `junos-eznc`
- `salt-proxy`

```bash
root@salt:~# salt-pip install napalm
```
Once done, we can proceed with the rest of the lab

## Part-1: Configuring the NAPALM Proxy Minion

As with the regular Minion, Proxy Minions have a dedicated configuration file, by default `/etc/salt/proxy`. In this 
file we can configure the same options as for the regular Minion, plus others specific to the Proxy Minions only: 
https://docs.saltstack.com/en/latest/ref/configuration/proxy.html.

One particular option to mention, inherited from the regular Minion, is `multiprocessing` which needs to be turned off 
for SSH-based Proxy Minions (i.e., when the communication channel is based on SSH); this option will ensure the Proxy 
Minion will be in multithreading mode instead.

As regular Minions run directly on the target device, they don't need the connection details, as they're implicit. Proxy 
Minions, on the other hand, as they connect to a remote device, they require the credentials in order to establish the 
connection. These connection details are typically provided through the Pillar system, as it provides more flexibility.

Say we have the following configuration on the Master, as in the previous Lab:

```yaml
pillar_roots:
  base:
    - /srv/salt/pillar
```

Under the `/srv/salt/pillar` directory we have the Pillar Top File, `top.sls`, which, for Proxy Minions must include 
a Pillar file with the connection details.

From your assigned machine, you can log into the devices we have prepared for this course, using SSH, e.g., `ssh 
apnic@router1`. The password is `APNIC2021`. `router1` is a Junos device. Test you are able to successfully log in:

```bash
root@group00:~# ssh apnic@router1
root@group00:~# ssh apnic@router1
Warning: Permanently added the ECDSA host key for IP address '172.22.0.18' to the list of known hosts.
Enter passphrase for key '/root/.ssh/id_ed25519': 
Password:
Last login: Tue Jan  5 12:32:24 2021 from 10.0.0.2
--- JUNOS 17.2R1.13 Kernel 64-bit  JNPR-10.3-20170523.350481_build
apnic> 
```

(If it asks for the password for the SSH key, simply press `Return`/`Enter`, then log in using the `APNIC2021` password)

This confirms the device is reachable and the credentials are correct.

Now, let's put these credentials in a Pillar file. The structure is below:

`/srv/salt/pillar/router1.sls`

```yaml
proxy:
  proxytype: napalm
  driver: junos
  host: router1
  username: apnic
  password: APNIC2021
```

Notice that the `proxytype` field has been specified as `napalm`, indicating that this is a NAPALM Proxy Minion. The 
`driver` field points to `junos`, as this is how NAPALM knows to select the right connection mechanism. The rest of the 
fields represent the credentials.

The Pillar Top File in this case, needs to be, at minimum:

`/srv/salt/pillar/top.sls`

```yaml
base:
  'router1':
    - router1
```

This structure ensures that the `router` Proxy Minion will have access to the data from `/srv/salt/pillar/router1.sls`.

This is all the preparation required, we can now start our first Proxy Minion, by executing:

```bash
root@salt:~# salt-proxy -l debug --proxyid router1
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.
[DEBUG   ] Reading configuration from /etc/salt/proxy

...
... snip ...
...
```

The log is more verbose than with the regular Minion, as there are more steps involved. Along the way, you will be able 
to notice debug logs such as:

```
[DEBUG   ] Setting up NAPALM connection
[DEBUG   ] [host None session 0x7f4af449f5f8] <SSHSession(session, initial daemon)> created: client_capabilities=<dict_keyiterator object at 0x7f4aef3193b8>
[DEBUG   ] starting thread (client mode): 0xef313dd8
[DEBUG   ] Local version/idstring: SSH-2.0-paramiko_2.7.1
[DEBUG   ] Remote version/idstring: SSH-2.0-OpenSSH_6.9
[INFO    ] Connected (version 2.0, client OpenSSH_6.9)
```

and

```
[INFO    ] [host router1 session-id 34633] Requesting 'ExecuteRpc'
[DEBUG   ] [host router1 session-id 34633] queueing <?xml version="1.0" encoding="UTF-8"?><nc:rpc xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:4fd86562-963c-4de3-8ac4-a83130707ee8"><get-system-uptime-information/></nc:rpc>
[DEBUG   ] [host router1 session-id 34633] Sync request, will wait for timeout=60
[DEBUG   ] [host router1 session-id 34633] Sending message
[INFO    ] [host router1 session-id 34633] Sending:
<?xml version="1.0" encoding="UTF-8"?><nc:rpc xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:4fd86562-963c-4de3-8ac4-a83130707ee8"><get-system-uptime-information/></nc:rpc>]]>]]>
```

These logs indicate the NETCONF connection being established and the RPC requests being executed to the device during 
the initial steps in order to gather the device facts (e.g., operating system, uptime, etc.); these details will be then 
available as Grains.


Once the setup is complete (connection established and the Grains collected), from a separate terminal window you will 
be able to start executing commands to confirm:

```bash
root@salt:~# salt router1 test.ping
router1:
    True
```

## Part-2: Configuring the Junos Proxy Minion

The Junos Proxy Minion abides to the same Proxy Minion standards, with the distinction that the device is managed 
through purely `junos-eznc` instead of NAPALM.

Let's update the `/srv/salt/pillar/router1.sls` file:

```yaml
proxy:
  proxytype: junos
  host: router1
  username: apnic
  password: APNIC2021
```

Notice that the `proxytype` field has been changed from `napalm` to `junos`, and there's no `driver` field anymore as the
Junos Proxy Module can only manage Juniper devices - unlike the NAPALM one which is able to manage many other platforms.

Nothing else needs to be update, and we can start the new Proxy Minion which is now Junos-based:

```bash
root@salt:~# salt-proxy -l debug --proxyid router1
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.
[DEBUG   ] Reading configuration from /etc/salt/proxy

...
... snip ...
...
```

The startup is exactly the same, and the intermediate logs are similar to the NAPALM Proxy too.

To confirm, we can once again execute a simple command such as:

```
root@salt:~# salt router1 test.ping
router1:
    True
```

Or display the available Grains:


```bash
root@salt:~# salt router1 grains.items
... snip ...
```


## Part-3: Configuring the Netmiko Proxy Minion

Similarly to the previous part, let's now switch to using the Netmiko Proxy Module, but updating the 
`/srv/salt/pillar/router1.sls` file:

```yaml
proxy:
  proxytype: netmiko
  device_type: juniper_junos
  host: router1
  username: apnic
  password: APNIC2021
```

The `proxytype` field has been changed to `netmiko`, of course, while `device_type` is the field required by Netmiko in 
order to identify the platform. See https://docs.saltstack.com/en/3000/ref/proxy/all/salt.proxy.netmiko_px.html for more 
details and what other platforms are now available to use.

Once again, we start the Proxy Minion in the same way as previously:

```bash
root@salt:~# salt-proxy -l debug --proxyid router1
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.
[DEBUG   ] Reading configuration from /etc/salt/proxy

...
... snip ...
...
```

You will notice in the debug logs that the setup is slightly different than previously, as Netmiko connects to the 
device via pure SSH, unlike NAPALM and Junos, which connect using NETCONF.

Once the Proxy Minion is up, we can start running commands:

```bash
root@salt:~# salt router1 test.ping
router1:
    True
```

And event execute CLI commands on the target device:

```bash
root@salt:~# salt router1 netmiko.send_command 'show version'
router1:

    Model: vmx
    Junos: 17.2R1.13
    JUNOS OS Kernel 64-bit  [20170523.350481_builder_stable_10]
    JUNOS OS libs [20170523.350481_builder_stable_10]
    JUNOS OS runtime [20170523.350481_builder_stable_10]
    JUNOS OS time zone information [20170523.350481_builder_stable_10]
    JUNOS network stack and utilities [20170601.185252_builder_junos_172_r1]
    JUNOS modules [20170601.185252_builder_junos_172_r1]
    JUNOS mx modules [20170601.185252_builder_junos_172_r1]
    JUNOS libs [20170601.185252_builder_junos_172_r1]
    JUNOS OS libs compat32 [20170523.350481_builder_stable_10]
    JUNOS OS 32-bit compatibility [20170523.350481_builder_stable_10]

...
... snip ...
...
```

---
**End of Lab**

---
