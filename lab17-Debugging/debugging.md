![](images/apnic_logo.png)
# LAB: Debugging Salt

As we've seen so far, Salt is capable to offer a large amount of features for managing systems. But all of these come 
with a cost, and have constantly increased Salt's complexity, and, implicitly, is has become more difficult to debug. 

The good news however is that Salt offers the tooling to help you debug it with ease.

The truth is, it almost never, or rarely happens for someone to get it right from the first try, and everyone from time 
to time runs into issues. For this, we need to know where to look after issues when debugging Salt.

## Part-1: Debugging the Master

Master issues are perhaps the easies to follow, as there's only one single place to start debugging from, and that is 
the Salt Master itself. The general rule of thumb is starting the Salt Master in debug mode and watch for errors logs.

One common problem is typos in the configuration file. The Master configuration file is interpreted as YAML, and 
therefore mistakes can happen frequently - for example incorrect indentation: update the `pillar_roots` in the
`/etc/salt/master` file and introduce two additional spaces at the beginning of the `pillar_roots` line:

`/etc/salt/master`

```yaml
  pillar_roots:
  base:
    - /srv/salt/pillar
```

Trying to start the Master it fails as:
```bash
salt-master -l debug
```

```bash
root@salt:~# salt-master -l debug
[DEBUG   ] Reading configuration from /etc/salt/master
[ERROR   ] Error parsing configuration file: /etc/salt/master - mapping values are not allowed here
  in "/etc/salt/master", line 3, column 15
[ERROR   ] Error parsing configuration file: /etc/salt/master - mapping values are not allowed here
  in "/etc/salt/master", line 3, column 15
```

This is one of the issues that are easy to detect as the Master doesn't start at all. A less obvious problem is when the 
Master starts fine, but issues occur due to incorrect configuration (although the YAML syntax is correct). For example, 
let's edit the same `pillar_roots` and point the configuration to an incorrect path:

```yaml
pillar_roots:
  base:
    - /path/to/incorrect/pillar/dir
```

The Master is able to start correctly, with no apparent issue:

```bash
root@salt:~# salt-master -l debug
[DEBUG   ] Reading configuration from /etc/salt/master

... snip ...
```

Let it run in the foreground, and open a separate terminal window, where we start a Proxy Minion:
```bash
salt-proxy --proxyid router1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid router1 -l debug
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.
[DEBUG   ] Reading configuration from /etc/salt/proxy

... snip ...

[ERROR   ] No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
[INFO    ] Proxy Minion Stopping the Salt ProxyMinion
[ERROR   ] No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
[INFO    ] Shutting down the Salt ProxyMinion
The Salt ProxyMinion is shutdown.
No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
```

The Proxy Minion is unable to start, as it's unable to pull the Pillar data.

The first command to the rescue when you see the `No proxy key found in pillar or opts` error message is check the 
output of the `salt-run pillar.show_pillar router1` command. A healthy output should look like:
```bash
salt-run pillar.show_pillar router1
```

```bash
root@salt:~# salt-run pillar.show_pillar router1
proxy:
    ----------
    proxytype:
        napalm
    driver:
        junos
    host:
        router1
    username:
        apnic
    password:
        APNIC2021
```

This command will return the entire Pillar data required for the `router` Proxy Minion, so likely you will have a longer 
output than just this; but, in order to have the Proxy Minion start correctly, at least the `proxy` block must be there.

Another category of common issues is related to Pillar compilation error. Right now, when starting the Proxy Minion for 
`router1` or when you've run `salt-run pillar.show_pillar router1`, you've seen the following errors:

```
[ERROR   ] Error on minion 'router1' http query: http://http_api:8888/
More Info:

[ERROR   ] error: [Errno -2] Name or service not known
[ERROR   ] Exception caught loading ext_pillar 'netbox':
  File "/usr/local/lib/python3.6/site-packages/salt/pillar/__init__.py", line 1146, in ext_pillar
    ext = self._external_pillar_data(pillar, val, key)
  File "/usr/local/lib/python3.6/site-packages/salt/pillar/__init__.py", line 1066, in _external_pillar_data
    ext = self.ext_pillars[key](self.minion_id, pillar, **val)
  File "/usr/local/lib/python3.6/site-packages/salt/pillar/netbox.py", line 93, in ext_pillar
    device_results["status"],

[CRITICAL] Pillar render error: Failed to load ext_pillar netbox: 'status'
```

This is because Salt is unable to execute the HTTP request to http://http_api:8888/; also, the NetBox External Pillar 
(see in the trace that the errors is thrown from the `/usr/local/lib/python3.6/site-packages/salt/pillar/netbox.py` 
file) complains. The first step when encountering this sort of errors is checking whether the external services are 
available. In this lab, both the HTTP API http://http_api:8888/ and the NetBox service have been turned off, and you can 
verify this by running, e.g., 
```bash
curl http://http_api:8888/
```

```bash
root@salt:~# curl http://http_api:8888/
curl: (6) Could not resolve host: http_api
root@salt:~#
```

As the service is unavailable or unreachable, Salt will, of course, fail to retrieve the data and throw errors.

When this Pillar data is used to compute the `proxy` block into the Pillar, this is the point you should start your 
debugging from: the external services must be reachable from your Salt Master.

## Part-2: Debugging Proxy Minions

One of the most frequent issue when working with Proxy Minions is related to the authentication Pillar. This is 
a critical starting point. As we've seen in the previous section, this can be strictly related to a misconfiguration on 
the Master side, or really just an issue with the Pillar data itself.

### Incorrect Pillar Top File

A common issue is in the _Pillar Top File_. We currently have the following structure:

`/srv/salt/pillar/top.sls`

```yaml
base:
  'router*':
    - junos
  'core*':
    - iosxr
  'spine*':
    - eos
  'leaf*':
    - ios
  'srv*':
    - ssh
    - proxies
```

This mapping references the `junos.sls`, `iosxr.sls`, ... `proxies.sls` files. This is why we sometimes may be tempted 
to have the following in the _Top File_ (which is incorrect):

```yaml
base:
  'router*':
    - junos.sls
  'core*':
    - iosxr.sls
  'spine*':
    - eos.sls
  'leaf*':
    - ios.sls
  'srv*':
    - ssh.sls
    - proxies.sls
```

This again leads to the `No proxy key found in pillar or opts` errors:

```bash
[ERROR   ] No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
[INFO    ] Proxy Minion Stopping the Salt ProxyMinion
[ERROR   ] No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
[INFO    ] Shutting down the Salt ProxyMinion
The Salt ProxyMinion is shutdown.
No proxy key found in pillar or opts for id router1. Check your pillar/opts configuration and contents.  Salt-proxy aborted.
```

### Incorrect Master

Another common issue is Master being unavailable or unreachable. To see this in action, let's add the following to the 
Proxy configuration file:

`/etc/salt/proxy`

```yaml
master: fake
```

When starting the Proxy Minion, it will fail with:

```
root@salt:~# salt-proxy --proxyid router1 -l debug
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.
[DEBUG   ] Reading configuration from /etc/salt/proxy

... snip ...

[DEBUG   ] Connecting to master. Attempt 1 of 1
[DEBUG   ] "fake" Not an IP address? Assuming it is a hostname.
[ERROR   ] DNS lookup or connection check of 'fake' failed.
[ERROR   ] Master hostname: 'fake' not found or not responsive. Retrying in 30 seconds
[ERROR   ] DNS lookup or connection check of 'fake' failed.
[ERROR   ] Master hostname: 'fake' not found or not responsive. Retrying in 30 seconds
```

The Minion will attempt to connect to the Master several times before it gives up.

In this particular scenario, the hostname was deliberately configured incorrectly. You may bump into issues when the 
address is correctly configured, but still unreachable; a common cause could be related to firewall issues: the Salt 
Master listens to connections on two TCP ports 4505 and 4506 and you will need to ensure these are open for connections.

Remove the `master: fake` line from `/etc/salt/proxy`. Another issue around the Proxy-Master communication is when the 
machine of the Master is reachable, but the Master itself not. To replicate this, go to the terminal where the Salt 
Master is running and stop it by Ctrl-C.

Try to start the Proxy Minion for `router1`:
```bash
salt-proxy --proxyid router1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid router1 -l debug


... snip ...


[DEBUG   ] Connecting the Minion to the Master URI (for the return server): tcp://172.22.0.3:4506
[DEBUG   ] Trying to connect to: tcp://172.22.0.3:4506
[DEBUG   ] salt.crypt.get_rsa_pub_key: Loading public key
[DEBUG   ] SaltReqTimeoutError, retrying. (1/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (2/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (3/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (4/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (5/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (6/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (7/7)
[DEBUG   ] Re-init ZMQ socket: Message timed out
[DEBUG   ] Trying to connect to: tcp://172.22.0.3:4506
[DEBUG   ] Closing AsyncZeroMQReqChannel instance
[ERROR   ] Error while bringing up minion for multi-master. Is master at salt responding?
```

We can see that the Minion tries to connect to the Master on `172.22.0.3` port `4506` but without success. As the error 
message suggests, when you're seeing this message you should probably verify that the Master is actually running and 
accepting connections.

Go back to the terminal where the Master is running, stop it by Ctrl-C, then re-start it by executing `salt-master -d`, 
to have it run in background, returning the prompt:
```bash
salt-master -d
```

```bash
root@salt:~# salt-master -d
root@salt:~#
```

### Incorrect or incomplete Proxy authentication

Correct the _Pillar Top File_ to the working state. Go back to the terminal where the Master was running previously, and 
start again the Master by running `salt-master -l debug`.

Now, edit `/srv/salt/pillar/junos.sls` and remove the line `proxytype: napalm`.

Starting the Proxy Minion would lead to the following errors:
```bash
salt-proxy --proxyid router1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid router1 -l debug

... snip ...
[CRITICAL] Unexpected error while connecting to salt
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1120, in _connect_minion
    yield minion.connect_master(failed=failed)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1064, in run
    yielded = self.gen.throw(*exc_info)
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1336, in connect_master
    yield self._post_master_init(master)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1070, in run
    yielded = self.gen.send(value)
  File "/usr/local/lib/python3.6/types.py", line 182, in send
    return self.__wrapped.send(val)
  File "/usr/local/lib/python3.6/site-packages/salt/metaproxy/proxy.py", line 118, in post_master_init
    fq_proxyname = self.opts["proxy"]["proxytype"]
KeyError: 'proxytype'
```

As the error would suggest, the key `proxytype` is missing. This is a critical component that Salt requires in order to 
be able to identify the Proxy module required to bring this Minion up. The error is the same for any Proxy module you 
would want to use.

Re-add the `proxytype: napalm` line and remove `driver: junos`. Starting the Proxy Minion, we will see a different 
error:
```bash
salt-proxy --proxyid router1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid router1 -l debug
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.

... snip ...

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1120, in _connect_minion
    yield minion.connect_master(failed=failed)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1064, in run
    yielded = self.gen.throw(*exc_info)
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1336, in connect_master
    yield self._post_master_init(master)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1070, in run
    yielded = self.gen.send(value)
  File "/usr/local/lib/python3.6/types.py", line 182, in send
    return self.__wrapped.send(val)
  File "/usr/local/lib/python3.6/site-packages/salt/metaproxy/proxy.py", line 183, in post_master_init
    proxy_init_fn(self.opts)
  File "/usr/local/lib/python3.6/site-packages/salt/proxy/napalm.py", line 210, in init
    NETWORK_DEVICE.update(salt.utils.napalm.get_device(opts))
  File "/usr/local/lib/python3.6/site-packages/salt/utils/napalm.py", line 341, in get_device
    _driver_ = provider_lib.get_network_driver(network_device.get("DRIVER_NAME"))
  File "/usr/local/lib/python3.6/site-packages/napalm/base/__init__.py", line 74, in get_network_driver
    raise ModuleImportError("Please provide a valid driver name.")
napalm.base.exceptions.ModuleImportError: Please provide a valid driver name.
```

The error comes from NAPALM this time, as the `driver` field is specific to the NAPALM Proxy module, and the message is 
self-explanatory.

Re-add the `driver: junos` line and update the username to `username: fake`:
```bash
salt-proxy --proxyid router1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid router1 -l debug
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.

... snip ...

[INFO    ] Authentication (password) failed.
[DEBUG   ] [host router1 session 0x7fe8a3acf588] Authentication failed.
[CRITICAL] Unexpected error while connecting to salt
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/jnpr/junos/device.py", line 1339, in open
    "use_filter": self._use_filter,
  File "/usr/local/lib/python3.6/site-packages/ncclient/manager.py", line 168, in connect
    return connect_ssh(*args, **kwds)
  File "/usr/local/lib/python3.6/site-packages/ncclient/manager.py", line 135, in connect_ssh
    session.connect(*args, **kwds)
  File "/usr/local/lib/python3.6/site-packages/ncclient/transport/ssh.py", line 362, in connect
    self._auth(username, password, key_filenames, allow_agent, look_for_keys)
  File "/usr/local/lib/python3.6/site-packages/ncclient/transport/ssh.py", line 464, in _auth
    raise AuthenticationError(repr(saved_exception))
ncclient.transport.errors.AuthenticationError: AuthenticationException('Authentication failed.',)

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1120, in _connect_minion
    yield minion.connect_master(failed=failed)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1064, in run
    yielded = self.gen.throw(*exc_info)
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1336, in connect_master
    yield self._post_master_init(master)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1070, in run
    yielded = self.gen.send(value)
  File "/usr/local/lib/python3.6/types.py", line 182, in send
    return self.__wrapped.send(val)
  File "/usr/local/lib/python3.6/site-packages/salt/metaproxy/proxy.py", line 183, in post_master_init
    proxy_init_fn(self.opts)
  File "/usr/local/lib/python3.6/site-packages/salt/proxy/napalm.py", line 210, in init
    NETWORK_DEVICE.update(salt.utils.napalm.get_device(opts))
  File "/usr/local/lib/python3.6/site-packages/salt/utils/napalm.py", line 350, in get_device
    network_device.get("DRIVER").open()
  File "/usr/local/lib/python3.6/site-packages/napalm/junos/junos.py", line 121, in open
    self.device.open()
  File "/usr/local/lib/python3.6/site-packages/jnpr/junos/device.py", line 1345, in open
    raise EzErrors.ConnectAuthError(self)
jnpr.junos.exception.ConnectAuthError: ConnectAuthError(router1)
```

The error message is again pretty simple to understand, as Salt complains that it is unable to authenticate to 
_router1_. This message however largely depends on the communication channel between Salt and the device, which is 
platform-specific; for example, for Juniper it is using NETCONF, while for Arista it is using HTTP requests over the 
Arista eAPI. To see the difference, make a similar change in the `/srv/salt/pillar/eos.sls` and start the Proxy Minion 
for `spine1`:
```bash
salt-proxy --proxyid spine1 -l debug
```

```bash
root@salt:~# salt-proxy --proxyid spine1 -l debug
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.

... spine ...

[DEBUG   ] Response: status:401, reason:Unauthorized
[DEBUG   ] Response content: b'Unable to authenticate user: Bad username/password combination'
[ERROR   ] Cannot connect to spine1 as fake.
[ERROR   ] Please check error: Unauthorized. b'Unable to authenticate user: Bad username/password combination'
[CRITICAL] Unexpected error while connecting to salt
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/napalm/eos/eos.py", line 142, in open
    self.device.run_commands(["show clock"], encoding="text")
  File "/usr/local/lib/python3.6/site-packages/pyeapi/client.py", line 743, in run_commands
    response = self._connection.execute(commands, encoding, **kwargs)
  File "/usr/local/lib/python3.6/site-packages/pyeapi/eapilib.py", line 550, in execute
    response = self.send(request)
  File "/usr/local/lib/python3.6/site-packages/pyeapi/eapilib.py", line 451, in send
    response_content))
pyeapi.eapilib.ConnectionError: Unauthorized. b'Unable to authenticate user: Bad username/password combination'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/salt/utils/napalm.py", line 350, in get_device
    network_device.get("DRIVER").open()
  File "/usr/local/lib/python3.6/site-packages/napalm/eos/eos.py", line 147, in open
    raise ConnectionException(py23_compat.text_type(ce))
napalm.base.exceptions.ConnectionException: Unauthorized. b'Unable to authenticate user: Bad username/password combination'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1120, in _connect_minion
    yield minion.connect_master(failed=failed)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1064, in run
    yielded = self.gen.throw(*exc_info)
  File "/usr/local/lib/python3.6/site-packages/salt/minion.py", line 1336, in connect_master
    yield self._post_master_init(master)
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1056, in run
    value = future.result()
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/concurrent.py", line 249, in result
    raise_exc_info(self._exc_info)
  File "<string>", line 4, in raise_exc_info
  File "/usr/local/lib/python3.6/site-packages/salt/ext/tornado/gen.py", line 1070, in run
    yielded = self.gen.send(value)
  File "/usr/local/lib/python3.6/types.py", line 182, in send
    return self.__wrapped.send(val)
  File "/usr/local/lib/python3.6/site-packages/salt/metaproxy/proxy.py", line 183, in post_master_init
    proxy_init_fn(self.opts)
  File "/usr/local/lib/python3.6/site-packages/salt/proxy/napalm.py", line 210, in init
    NETWORK_DEVICE.update(salt.utils.napalm.get_device(opts))
  File "/usr/local/lib/python3.6/site-packages/salt/utils/napalm.py", line 367, in get_device
    raise napalm_base.exceptions.ConnectionException(base_err_msg)
napalm.base.exceptions.ConnectionException: Cannot connect to spine1 as fake.
```

This is the error message you would get for authentication failure with Arista switches.

## Part-3: Debugging Jinja and SLS

Jinja can be particularly difficult and unpleasant to debug. But, luckily, Salt has the batteries included to make our 
lives easier. As we've seen, any Jinja template rendered through Salt is powered up by providing access to various 
Salt-specifics such as Grains, Pillars, and - perhaps most importantly - any Execution Function.

Such is the case with a group of functions from the `log` Execution Module: `log.debug`, `log.info`, `log.warning`, 
`log.error` - which would push a logging message at the specific level the function invoked is designed.

Update `/srv/salt/pillar/junos.sls` and re-configure `username: apnic` in order to have the correct authentication.

Start the Proxy Minion for `router1` in daemon mode, to have it run in background, but logging at debug level:
```bash
salt-proxy -l debug --proxyid router1
```
```bash
root@salt:~# salt-proxy -l debug --proxyid router1 -d
[DEBUG   ] Reading configuration from /etc/salt/proxy
[INFO    ] Processing `log_handlers.sentry`
[DEBUG   ] Grains refresh requested. Refreshing grains.

... snip ...

[DEBUG   ] Loading static grains from /etc/salt/proxy.d/router1/grains
[DEBUG   ] LazyLoaded zfs.is_supported
[DEBUG   ] No 'sentry_handler' key was found in the configuration
[INFO    ] The `log_handlers.sentry.setup_handlers()` function returned `False` which means no logging handler was configured on purpose. Continuing...
[DEBUG   ] Configuration file path: /etc/salt/proxy
root@salt:~#
```

Immediately after this, start watching the Proxy Minion logging file:
```bash
tail -f /var/log/salt/proxy
```

```bash
root@salt:~# tail -f /var/log/salt/proxy
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :328 ][DEBUG   ][18711] {'out': [], 'result': True, 'comment': ''}
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :334 ][DEBUG   ][18711] Comparing to:
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :335 ][DEBUG   ][18711] {'synchronized': False}
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :249 ][DEBUG   ][18711] Comparing dict to list (of dicts?)
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :343 ][DEBUG   ][18711] Result of comparison: False
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :355 ][DEBUG   ][18711] NAPALM beacon generated the events:
2021-02-17 18:51:49,254 [salt.loaded.int.beacons.napalm_beacon                              :356 ][DEBUG   ][18711] []
2021-02-17 18:51:54,259 [ncclient.transport.ssh                                             :1819][DEBUG   ][18711] Sending global request "keepalive@lag.net"
2021-02-17 18:51:59,265 [ncclient.transport.ssh                                             :1819][DEBUG   ][18711] Sending global request "keepalive@lag.net"
2021-02-17 18:52:04,271 [ncclient.transport.ssh                                             :1819][DEBUG   ][18711] Sending global request "keepalive@lag.net"
2021-02-17 18:52:09,277 [ncclient.transport.ssh                                             :1819][DEBUG   ][18711] Sending global request "keepalive@lag.net"
2021-02-17 18:52:14,284 [ncclient.transport.ssh                                             :1819][DEBUG   ][18711] Sending global request "keepalive@lag.net

... snip ...
```

You are going to notice there already is some activity at the DEBUG level.

In an SLS file, let's put the following instructions:

`/srv/salt/debug.sls`

```sls
{%- do salt.log.debug('Test debug message') %}
{%- do salt.log.info(grains.model) %}
{%- do salt.log.warning(grains.serial) %}
{%- do salt.log.error(salt.system.get_system_date()) %}

version: {{ grains.version }}
```

This pushes a few log lines at various logging levels, displaying various values such as the `model` and the `serial` 
Grains, and the value returned by the `system.get_system_date` Execution function being called.

To render this SLS file, we can use the `slsutil.renderer` helper, which helps us preview the contents of an SLS file 
after being interpreted:
```bash
salt router1 slsutil.renderer salt://debug.sls
```

```bash
root@salt:~# salt router1 slsutil.renderer salt://debug.sls
router1:
    ----------
    version:
        17.2R1.13
```

The file only provides a single key-value pair. In the logs we can notice the following lines resulted during the 
rendering of the `/srv/salt/debug.sls` file:


```
2021-02-17 18:59:39,434 [salt.loader.salt.int.module.logmod                                 :42  ][DEBUG   ][18711] Test debug message
2021-02-17 18:59:39,434 [salt.loader.salt.int.module.logmod                                 :50  ][INFO    ][18711] VMX
2021-02-17 18:59:39,434 [salt.loader.salt.int.module.logmod                                 :58  ][WARNING ][18711] VM602D17DB41
2021-02-17 18:59:39,434 [salt.loader.salt.int.module.logmod                                 :66  ][ERROR   ][18711] Wed 02/17/2021
```

## Part-4: Debugging Extension Modules using ISalt

When writing our own extension modules, inevitably we're likely going to make use of the Salt-specific magic variables 
such as `__opts__`, `__grains__`, `__salt__` and so on. While understanding their scope and usability may be trivial, 
the data they have access to may not always be obvious and almost always it highly depends on the environment the Master 
and / or Minions are running. This is why a third-party tool such as [ISalt](https://isalt.dev) can help alleviate these 
difficulties.

ISalt is an enhanced IPython console that provides you with the usual Salt dunders directly for you to use. It provides 
a few different modes you can use if with: Master, Minion, Proxy Minion, Masterless Minion, and Salt SProxy. The 
Salt-specific dunders usually have different meaning, scope, and values depending on each mode: `__opts__` on the Master 
refers to the configuration options for the Salt Master, while `__opts__` on the Minion refers to the configuration 
options for the Salt Minion; `__salt__` on the Master refers to the Runners, while `__salt__` on the Minion refers to 
the Execution Modules and so on.

### ISalt Master
To install you can use pip
```
pip install isalt
```

when we want to have the dunders for the Master side loaded, we can run:
```bash
isalt --master
```

```bash
root@salt:~# isalt --master
 __       _______.     ___       __      .___________.
|  |     /       |    /   \     |  |     |           |
|  |    |   (----`   /  ^  \    |  |     `---|  |----`
|  |     \   \      /  /_\  \   |  |         |  |
|  | .----)   |    /  _____  \  |  `----.    |  |
|__| |_______/    /__/     \__\ |_______|    |__|




           Role: Master
        Salt version: 3001.1
       IPython version: 7.16.1

In [1]:
```

This is going to open the IPython console with the variables pre-loaded:

```bash
In [1]: __opts__['__role']
Out[1]: 'master'

In [2]: __salt__['test.sleep'](1)
1
Out[2]: True

In [3]: __salt__['salt.cmd']('test.ping')
Out[3]: True

In [4]: __salt__['salt.cmd']('example.test')
Out[4]: True

In [5]: __salt__['salt.cmd']('example.hello')
Out[5]: 'Hello stranger'

In [6]: __salt__['salt.cmd']('example.os_name')
Out[6]: 'Debian'
```

Both `__opts__` and `__salt__` are directly available, with the data pre-populated and we are able to invoke various 
Runners in an interactive manner.

Exit the console by typing `exit`.

### ISalt Proxy Minion

To enter in the Proxy Minion mode, we need to pass the `--proxy` CLI flag, as well as providing the ID of the Minion (or 
device name) we want to load the data for; this is important as the values are highly dependent on the device we want to 
load the variables for: not only that Grains and Pillars are highly dependent, but the Execution Modules may differ from 
one device to another (if we configure it like that):
```bash
isalt -e /etc/salt/proxy --proxy --minion-id router1
```

```bash
root@salt:~# isalt -e /etc/salt/proxy --proxy --minion-id router1
 __       _______.     ___       __      .___________.
|  |     /       |    /   \     |  |     |           |
|  |    |   (----`   /  ^  \    |  |     `---|  |----`
|  |     \   \      /  /_\  \   |  |         |  |
|  | .----)   |    /  _____  \  |  `----.    |  |
|__| |_______/    /__/     \__\ |_______|    |__|




           Role: Proxy
        Salt version: 3001.1
       IPython version: 7.16.1

In [1]:
```

Immediately after getting in this mode, we can start executing various requests:

```bash
In [1]: __opts__['__role']
Out[1]: 'minion'

In [2]: __grains__['id']
Out[2]: 'router1'

In [3]: __grains__['vendor'], __grains__['model'], __grains__['version']
Out[3]: ('Juniper', 'VMX', '17.2R1.13')

In [4]: __salt__['net.arp']()
Out[4]:
{'out': [{'interface': 'fxp0.0',
   'mac': '52:55:0A:00:00:02',
   'ip': '10.0.0.2',
   'age': 628.0},
  {'interface': 'em1.0',
   'mac': '52:54:00:58:42:00',
   'ip': '128.0.0.16',
   'age': None}],
 'result': True,
 'comment': ''}

In [5]: __pillar__
Out[5]:
{'proxy': {'proxytype': 'napalm',
  'driver': 'junos',
  'host': 'router1',
  'username': 'apnic',
  'password': 'APNIC2021'},
 '_errors': ["Failed to load ext_pillar netbox: 'status'"]}

In [6]:
```

### ISalt with Salt SProxy

For the previous command to work, the Salt key of `router1` must have been accepted previously, otherwise it will not 
work. In that case, there are times when we'd probably like to execute Salt tasks without this overhead, and this is 
exactly where Salt SProxy is able to help:
```bash
isalt --sproxy
```

```bash
root@salt:~# isalt --sproxy
 __       _______.     ___       __      .___________.
|  |     /       |    /   \     |  |     |           |
|  |    |   (----`   /  ^  \    |  |     `---|  |----`
|  |     \   \      /  /_\  \   |  |         |  |     
|  | .----)   |    /  _____  \  |  `----.    |  |     
|__| |_______/    /__/     \__\ |_______|    |__|     




           Role: Sproxy
        Salt version: 3001.1
       IPython version: 7.16.1

In [1]:
```

Starting ISalt with `--sproxy`, ISalt provides a feature, named `sproxy` which gives us access to directly invoke 
commands:

```bash
In [1]: sproxy('router*', salt_function='test.ping', static=True)
Out[1]: {'router2': True, 'router1': True}

In [2]: sproxy('spine1', salt_function='net.lldp', static=True)
Out[2]:
{'spine1': {'out': {'Ethernet2': [{'parent_interface': 'Ethernet2',
     'remote_port': '"Gi0/0/0/4"',
     'remote_port_description': '',
     'remote_system_name': 'core1',
     'remote_system_description': 'Cisco IOS XR Software, Version 6.0.0[Default]\nCopyright (c) 2015 by Cisco Systems, Inc., IOS XRv Series',
     'remote_chassis_id': '02:3E:CD:56:A4:06',
     'remote_system_capab': ['router'],
     'remote_system_enable_capab': ['router']}],
   'Ethernet3': [{'parent_interface': 'Ethernet3',
     'remote_port': '"Gi0/0/0/4"',
     'remote_port_description': '',
     'remote_system_name': 'core2',
     'remote_system_description': 'Cisco IOS XR Software, Version 6.0.0[Default]\nCopyright (c) 2015 by Cisco Systems, Inc., IOS XRv Series',
     'remote_chassis_id': '02:66:B4:6C:A4:06',
     'remote_system_capab': ['router'],
     'remote_system_enable_capab': ['router']}],

... snip ...

  'result': True,
  'comment': ''}}

In [3]: sproxy('srv*', salt_function='system.get_system_time', static=True)
Out[3]:
{'srv4': '12:49:54 PM',
 'srv3': '12:49:54 PM',
 'srv2': '12:49:54 PM',
 'srv1': '12:49:54 PM'}
```

The output can be stored into various variables then used in different contexts.

--
**End of Lab**

---
