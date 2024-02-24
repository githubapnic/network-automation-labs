![](images/apnic_logo.png)
# LAB: Automating using Salt, without running Proxy Minions

For this lab, as we want to showcase the usage of Salt but without having to run Proxy Minions for the network devices, we will stop the Proxy Minions for most of the topology - the only ones running will be those for the `leaf` switches, as typically those are the ones we interact with most of the time. In other word, the Proxy Minions for `leaf1`, `leaf2`, `leaf3` and `leaf4` are running, but the rest of the devices will be managed through _salt-sproxy_.

_salt-sproxy_ is pre-installed, so we can start automating straight away.

## Part-1: Getting started with Salt SProxy

First thing we need to decide when stating using _salt-sproxy_ is where we define the list of devices we want to manage. In this lab, for simplicity, and avoid external dependencies, we will list the devices into a file. This is called the _Roster_. In the Master configuration file we need to provide this information:

```bash
grep roster /etc/salt/master
```

<pre>
root@salt:~# grep roster /etc/salt/master
roster: file
roster_file: /etc/salt/roster
</pre>

This way, Salt will know that the Roster is defined as a file, and it is located at `/etc/salt/roster`.

The Roster file has the following contents:

```bash
cat /etc/salt/roster
```

<pre>
router1:
  grains:
    role: router
router2:
  grains:
    role: router
core1:
  grains:
    role: core
core2:
  grains:
    role: core

{%- for i in range(1,5) %}
spine{{ i }}:
  grains:
    role: spine
{%- endfor %}

{%- for i in range(1,5) %}
leaf{{ i }}:
  grains:
    role: leaf
{%- endfor %}
</pre>

For the beginning, let's analyze the first 3 lines:

```bash
head -3 /etc/salt/roster
```

<pre>
router1:
  grains:
    role: router
</pre>

This simple YAML structure points out that we want to manage the device `router1`. Underneath its key, there's `grains`, where we can defined static Grains to be assigned to it. In this case, the `role` Grain has the value `router`. As in the usual Salt, these Grains can be used for targeting, or referenced in various Jinja / SLS templates.

By default, the Roster file is interpreted as SLS, and therefore we can benefit from all the advantages. Thanks to this, we can have `for` loops in order to have the file more condensed:

```bash
grep "for i" -A 4 /etc/salt/roster
```

<pre>
{%- for i in range(1,5) %}
spine{{ i }}:
  grains:
    role: spine
{%- endfor %}
</pre>

The block above will be rendered and interpreted as the following YAML structure:

<pre>
spine1:
  grains:
    role: spine
spine2:
  grains:
    role: spine
spine3:
  grains:
    role: spine
spine4:
  grains:
    role: spine
</pre>

The advantage may not be immediately clear in this example, but if we were to have hundreds of spine switches, the `for` loop remains the same.

In other words, for clarity, the Roster file is interpreted as:

<pre>
router1:
  grains:
    role: router
router2:
  grains:
    role: router
core1:
  grains:
    role: core
core2:
  grains:
    role: core
spine1:
  grains:
    role: spine
spine2:
  grains:
    role: spine
spine3:
  grains:
    role: spine
spine4:
  grains:
    role: spine
leaf1:
  grains:
    role: leaf
leaf2:
  grains:
    role: leaf
leaf3:
  grains:
    role: leaf
leaf4:
  grains:
    role: leaf
</pre>

This is our entire topology.

Everything else remains the same as previously. The authentication credentials are in the Pillar, as configured for the running Proxy Minions. As a reminder, they were configured like this:

```bash
cat /srv/salt/pillar/top.sls
```

<pre>
base:
  'router*':
    - junos
  'core*':
    - iosxr
  'spine*':
    - eos
  'leaf*':
    - ios
</pre>

```bash
cat /srv/salt/pillar/junos.sls
```

<pre>
proxy:
  proxytype: napalm
  driver: junos
  host: {{ opts.id }}
  username: apnic
  password: APNIC2021
</pre>

All of these are equally available now, when running through salt-sproxy, without having Proxy Minions running.

Running the following command, you can check what Minions are connected to this Master:

```bash
salt-key -L
```

<pre>
root@salt:~# salt-key -L
Accepted Keys:
core1
core2
leaf1
leaf2
leaf3
leaf4
router1
router2
spine1
spine2
spine3
spine4
Denied Keys:
Unaccepted Keys:
Rejected Keys:
</pre>

**Note**: There may be a python error, but the command completes succesfully

<pre>
  root@salt:~# salt-key -L
/usr/local/lib/python3.6/site-packages/requests/__init__.py:91: RequestsDependencyWarning: urllib3 (1.26.18) or chardet (3.0.4) doesn't match a supported version!
  RequestsDependencyWarning)
</pre>

This can be fixed by installing the requests module for python, using the following command:

```bash
pip3 install requests
```

As we no longer need all of them, let's remove the key of the routers, cores and spines:

```bash
salt-key -d router*
```

<pre>
root@salt:~# salt-key -d router*
The following keys are going to be deleted:
Accepted Keys:
router1
router2
Proceed? [N/y] y
Key for minion router1 deleted.
Key for minion router2 deleted.
</pre>

Same for `salt-key -d core*` and `salt-key -d spine*`.

Only the keys for the leaf switches are still accepted now:

```bash
salt-key -L
```

<pre>
root@salt:~# salt-key -L
Accepted Keys:
leaf1
leaf2
leaf3
leaf4
Denied Keys:
Unaccepted Keys:
Rejected Keys:
</pre>

_Note_: this cleanup is not absolutely needed, and we're only doing it because we've previously had Proxy Minions connected to this Master, that we no longer need. In a fresh _salt-sproxy_ environment, this extra step is not required.

At this point, we're sure that only leaf switches have Proxy Minions running, and can start managing everything else through _salt-sproxy_:

If we'd be running commands against the routers, cores or spines, we'd get the following error:

```bash
salt router1 test.ping 2> /dev/null
```

<pre>
root@salt:~# salt router1 test.ping
No minions matched the target. No command was sent, no jid was assigned.
ERROR: No return received
</pre>

But we can execute using:

```bash
salt-sproxy router1 test.ping
```

<pre>
root@salt:~# salt-sproxy router1 test.ping
router1:
    True
</pre>

Notice that the execution takes longer than when using the `salt` command. To understand why, run the same command, in debug mode:

```bash
salt-sproxy router1 test.ping -l debug
```

What do you notice?

It is expected, by design, that it takes longer, as when executing through `salt-sproxy` the connection is initiated on demand. In opposition, when having running Proxy Minions, the connection is established only on startup, then the Minion maintains it and it only passe the commands and the results back and forth.

That also means that the execution time depends on the way _salt-sproxy_ connects to the device: `router1` is managed via NETCONF, which is over an SSH channel, and therefore slower than, for example, a platform managed through HTTP request, such as Arista EOS:

```bash
time salt-sproxy router1 test.ping
```

<pre>
root@salt:~# time salt-sproxy router1 test.ping
router1:
    True

real	0m7.624s
user	0m5.645s
sys	0m0.756s
</pre>

```bash
time salt-sproxy spine1 test.ping
```

<pre>
root@salt:~# time salt-sproxy spine1 test.ping
spine1:
    True

real	0m4.820s
user	0m1.461s
sys	0m0.263s
</pre>

In general, everything we've done with Salt is available through _salt-sproxy_ as well. To prove this, execute a few commands:

```bash
salt-sproxy core* net.lldp
```

```bash
salt-sproxy router* route.show 0.0.0.0/0
```

But we also have the Proxy Minions for the leaf switches available. Thanks to the `use_existing_proxy: true` option configured in `/etc/salt/master`, _salt-sproxy_ will also attempt to run commands on the running Proxy Minions:

```bash
salt-sproxy -G role:leaf test.ping
```

<pre>
root@salt:~# salt-sproxy -G role:leaf test.ping
leaf2:
    True
leaf3:
    True
leaf1:
    True
leaf4:
    True

real	0m2.035s
user	0m1.356s
sys	0m0.131s
</pre>

The execution time is small, as _salt-sproxy_ simply passes the command to the Proxy Minions, which are already connected, so it doesn't spend much time for this.

## Part-2: Event-driven automation using salt-sproxy

By adding the `event: true` configuration option, while having a Salt Master running, we can ensure that _salt-sproxy_ will place events on the bus as we execute commands. For example, running the simplest command possible, e.g., `salt-sproxy spine1 test.ping`, there will be a number of events seen on the Salt bus. 

Type the following to watch the Salt event bus (`salt-run state.event pretty=True`):

```bash
salt-run state.event pretty=True
```

Open another terminal and type:

```bash
salt-sproxy spine1 test.ping
```

Return to the Salt event bus terminal window to review the output.

<pre>
proxy/runner/20210121173613864214/new	{
    "_stamp": "2021-01-21T17:36:13.865274",
    "arg": [],
    "fun": "test.ping",
    "jid": "20210121173613864214",
    "minions": [
        "spine1"
    ],
    "tgt": "spine1",
    "tgt_type": "glob",
    "user": "root"
}
proxy/runner/20210121173613864214/ret/spine1	{
    "_stamp": "2021-01-21T17:36:16.806624",
    "fun": "test.ping",
    "fun_args": [],
    "id": "spine1",
    "jid": "20210121173613864214",
    "retcode": 0,
    "return": true,
    "success": true
}
</pre>

The structure of these events respect exactly the one we've seen previously when executing commands via the `salt` CLI. The only difference is the tag, which starts with `proxy/runner/` instead of `salt/job/`. Everything else remains the same, so if you have Reactors configured to the normal Salt tags, all you have to do it to make sure they also match `proxy/runner/*/new` and/or `proxy/runner/*/ret/*` pattern(s).

For example, previously, in _Lab 11_, we had the following configuration:

<pre>
reactor:
  - 'salt/job/*/ret/*':
    - salt://reactor/test.sls
</pre>

This has matched job returns. To make sure that the `salt://reactor/test.sls` Reactor is invoked for job returns when executing through _salt-sproxy_, this configuration becomes:

<pre>
reactor:
  - 'salt/job/*/ret/*':
    - salt://reactor/test.sls
  - 'proxy/runner/*/ret/*':
    - salt://reactor/test.sls
</pre>

With both Reactors in place, we can be sure that `salt://reactor/test.sls` is invoked in a mixed environment (i.e., with both running Proxy Minions, but also when managing through _salt-sproxy_ only). Everything else stays the same as always.

Things are a little bit different inside the Reactor SLS. Let's look again at the Reactor SLS we've used to backup the configuration in response to a `CONFIGURATION_COMMIT_COMPLETED` notification from napalm-logs. As a refresher, the reaction is configured as:

Return to the terminal window where you can type commands.

```bash
 grep "reactor:" -A 3 /etc/salt/master
```

<pre>
reactor:
  - 'napalm/syslog/*/CONFIGURATION_COMMIT_COMPLETED/*':
    - salt://reactor/bkup.sls
</pre>

The `salt://reactor/bkup.sls` was defined as:

```bash
cat /srv/salt/reactor/bkup.sls
``` 

<pre>
Backup config:
  local.net.save_config:
    - tgt: {{ data.host }}
    - kwarg:
        source: running
        path: /tmp/{{ data.host }}.conf
</pre>

This Reactor SLS is using the `local` client. But, as our devices are managed through _salt-sproxy_ instead, the `local` client is unavailable, so this structure needs to be slightly changed to:

```bash
sed -i 's/local.net.save_config/runner.proxy.execute/' /srv/salt/reactor/bkup.sls
cat /srv/salt/reactor/bkup.sls
```

<pre>
Backup config:
  <strong>runner.proxy.execute</strong>:
    - tgt: {{ data.host }}
    - kwarg:
        salt_function: net.save_config
        source: running
        path: /tmp/{{ data.host }}.conf
</pre>

We have `runner.proxy.execute` instead of `local.net.save_config`, and the function to be invoked is specified under `kwarg`, as `salt_function: net.save_config`.

`runner.proxy.execute` instructs the Reactor to invoke the `proxy.execute` Runner, which is the actual core of _salt-sproxy_. The `proxy` Runner is shipped as part of the _salt-sproxy_ package, so Salt needs to be made aware of this, in order to find it. We can do so by running:

```bash
salt-run saltutil.sync_all
```

<pre>
root@salt:~# salt-run saltutil.sync_all
cache:
clouds:
engines:
executors:
    - executors.ssh
fileserver:
grains:
modules:
output:
pillar:
proxymodules:
    - proxy.ssh
queues:
renderers:
returners:
runners:
    - runners.__init__
    - runners.proxy
sdb:
serializers:
states:
thorium:
tokens:
tops:
utils:
wheel:
</pre>

With these changes, let's apply a configuration change on `router1` and then watch the Salt event bus:

```bash
salt-sproxy router1 net.load_config text='set system name-server 1.1.1.1'
```

<pre>
root@salt:~# salt-sproxy router1 net.load_config text='set system name-server 1.1.1.1'
router1:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit system]
        +  name-server {
        +      1.1.1.1;
        +  }
    loaded_config:
    result:
        True
</pre>

We will see the following sequence of events:

<pre>
proxy/runner/20210121180946260516/new	{
    "_stamp": "2021-01-21T18:09:46.261540",
    "arg": [
        {
            "__kwarg__": true,
            "text": "set system name-server 1.1.1.1"
        }
    ],
    "fun": "net.load_config",
    "jid": "20210121180946260516",
    "minions": [
        "router1"
    ],
    "tgt": "router1",
    "tgt_type": "glob",
    "user": "root"
}
napalm/syslog/junos/CONFIGURATION_COMMIT_REQUESTED/router1	{
    "_stamp": "2021-01-21T18:09:55.174014",
    "error": "CONFIGURATION_COMMIT_REQUESTED",
    "facility": 23,
    "host": "router1",
    "ip": "172.22.1.1",
    "message_details": {
        "date": "Jan 21",
        "facility": 23,
        "host": "router1",
        "hostPrefix": null,
        "message": "User 'apnic' requested 'commit' operation (comment: none)",
        "pri": "189",
        "processId": "23449",
        "processName": "mgd",
        "severity": 5,
        "tag": "UI_COMMIT",
        "time": "18:09:52"
    },
    "os": "junos",
    "severity": 5,
    "timestamp": 1611252592,
    "yang_message": {
        "users": {
            "user": {
                "apnic": {
                    "action": {
                        "comment": "none",
                        "requested_commit": true
                    }
                }
            }
        }
    },
    "yang_model": "NO_MODEL"
}
napalm/syslog/junos/CONFIGURATION_COMMIT_COMPLETED/router1	{
    "_stamp": "2021-01-21T18:09:55.707080",
    "error": "CONFIGURATION_COMMIT_COMPLETED",
    "facility": 23,
    "host": "router1",
    "ip": "172.22.1.1",
    "message_details": {
        "date": "Jan 21",
        "facility": 23,
        "host": "router1",
        "hostPrefix": null,
        "message": "commit complete",
        "pri": "188",
        "processId": "23449",
        "processName": "mgd",
        "severity": 4,
        "tag": "UI_COMMIT_COMPLETED",
        "time": "18:09:52"
    },
    "os": "junos",
    "severity": 4,
    "timestamp": 1611252592,
    "yang_message": {
        "system": {
            "operations": {
                "commit_complete": true
            }
        }
    },
    "yang_model": "NO_MODEL"
}
proxy/runner/20210121180946260516/ret/router1	{
    "_stamp": "2021-01-21T18:09:55.893811",
    "fun": "net.load_config",
    "fun_args": [
        {
            "__kwarg__": true,
            "text": "set system name-server 1.1.1.1"
        }
    ],
    "id": "router1",
    "jid": "20210121180946260516",
    "retcode": 0,
    "return": {
        "already_configured": false,
        "comment": "",
        "diff": "[edit system]\n+  name-server {\n+      1.1.1.1;\n+  }",
        "loaded_config": "",
        "result": true
    },
    "success": true
}
salt/run/20210121180955946790/new	{
    "_stamp": "2021-01-21T18:09:55.948572",
    "fun": "runner.proxy.execute",
    "fun_args": [
        "router1",
        {
            "path": "/tmp/router1.conf",
            "salt_function": "net.save_config",
            "source": "running"
        }
    ],
    "jid": "20210121180955946790",
    "user": "Reactor"
}
proxy/runner/20210121180955964808/new	{
    "_stamp": "2021-01-21T18:09:55.965837",
    "arg": [
        {
            "__kwarg__": true,
            "path": "/tmp/router1.conf",
            "source": "running"
        }
    ],
    "fun": "net.save_config",
    "jid": "20210121180955964808",
    "minions": [
        "router1"
    ],
    "tgt": "router1",
    "tgt_type": "glob",
    "user": "root"
}
proxy/runner/20210121180955964808/ret/router1	{
    "_stamp": "2021-01-21T18:09:59.333788",
    "fun": "net.save_config",
    "fun_args": [
        {
            "__kwarg__": true,
            "path": "/tmp/router1.conf",
            "source": "running"
        }
    ],
    "id": "router1",
    "jid": "20210121180955964808",
    "retcode": 0,
    "return": {
        "comment": "running config saved to /tmp/router1.conf",
        "out": "/tmp/router1.conf",
        "result": true
    },
    "success": true
}
</pre>

In this, we notice:

- `proxy/runner/20210121180946260516/new` the job creation event, when we requested _salt-sproxy_ to apply the configuration change.
- `napalm/syslog/junos/CONFIGURATION_COMMIT_REQUESTED/router1` and `napalm/syslog/junos/CONFIGURATION_COMMIT_COMPLETED/router1` as the napalm-logs events - again, seen on the bus before   Salt replies, which is a big plus.
- `proxy/runner/20210121180946260516/ret/router1` as the job return event, with the result printed on the command line.
- `proxy/runner/20210121180955964808/new` and `proxy/runner/20210121180955964808/ret/router1`: the job creation and return events for the reaction to the napalm-logs notifications. They are similarly prefixes with the `proxy/runner/` tag, as they are equally managed through _salt-sproxy_.

## Part-3: Using the REST API with salt-sproxy

While for the event-driven methodologies there are some slight changes required, in order to make _salt-sproxy_ calls through the Salt API. The only difference is that instead of starting the API using `salt-api`, we will do:

```bash
salt-sapi -l debug
```

<pre>
root@salt:~# salt-sapi -l debug
...

[INFO    ] [21/Jan/2021:18:32:18] ENGINE Listening for SIGTERM.
[INFO    ] [21/Jan/2021:18:32:18] ENGINE Listening for SIGHUP.
[INFO    ] [21/Jan/2021:18:32:18] ENGINE Listening for SIGUSR1.
[INFO    ] [21/Jan/2021:18:32:18] ENGINE Bus STARTING
[INFO    ] [21/Jan/2021:18:32:18] ENGINE Serving on http://0.0.0.0:8080
[INFO    ] [21/Jan/2021:18:32:18] ENGINE Bus STARTED
</pre>

Open a separate terminal window where we'll be executing commands. For starters, querying the main page:

```bash
curl http://0.0.0.0:8080
```

<pre>
root@salt:~# curl http://0.0.0.0:8080
{"return": "Welcome", "clients": ["local", "local_async", "local_batch", "local_subset", "runner", "runner_async", "sproxy", "sproxy_async", "ssh", "wheel", "wheel_async"]}
</pre>

In addition to the what we've seen before, in _Lab 11_, there are two new clients available: `sproxy` and `sproxy_async`. These are the ones we'll be using to make queries via _salt-sproxy_.

From the previous lab, again, we have executed the following request:

```bash
curl http://0.0.0.0:8080/run -d eauth=auto -d username=test-usr -d password=test -d client=local -d tgt=router1 -d fun=test.ping
```

<pre>
root@salt:~# curl http://0.0.0.0:8080/run -d eauth=auto -d username=test-usr -d password=test -d client=local -d tgt=router1 -d fun=test.ping
{"return": [{"router1": true}]}
</pre>

This used the `local` client. As mention above, in _Part-2_, the local client is not available for this operation, so we switch to using `sproxy` instead:

```bash
curl http://0.0.0.0:8080/run -d eauth=auto -d username=test-usr -d password=test -d client=sproxy -d tgt=router1 -d fun=test.ping
```

<pre>
root@salt:~# curl http://0.0.0.0:8080/run -d eauth=auto -d username=test-usr -d password=test -d client=sproxy -d tgt=router1 -d fun=test.ping
{"return": [{"router1": true}]}
</pre>

Besides the `client` argument nothing else changes, and the output has the same format as well. For the webhooks and everything else through the Salt API, all stays the same as always.

## Part-4: (Optional) Salt SProxy as a replacement for Salt SSH

_salt-sproxy_ is flexible enough to be able to manage any type of device, as long as there is a Proxy Module for it. 
The package is provided with a Proxy Module named `ssh` which can be used to make SSH connections. As the modules are 
external (_salt-sproxy_ is just a plugin), we need to make Salt aware of these modules:

```bash
root@salt:~# salt-run saltutil.sync_all
cache:
clouds:
engines:
executors:
    - executors.ssh
fileserver:
grains:
modules:
output:
pillar:
proxymodules:
    - proxy.ssh
queues:
renderers:
returners:
runners:
    - runners.__init__
    - runners.proxy
sdb:
serializers:
states:
thorium:
tokens:
tops:
utils:
wheel:
```

In our environment, there are 4 available servers, named `srv1`, `srv2`, `srv3`, and `srv4`. You can log into those, by 
using the `/etc/salt/ssh_key` SSH key (the password is `APNIC2021`):

```
root@salt:~# ssh -i /etc/salt/ssh_key root@srv1
Enter passphrase for key '/etc/salt/ssh_key':
Linux srv1 5.4.0-47-generic #51~18.04.1-Ubuntu SMP Sat Sep 5 14:35:50 UTC 2020 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Jan 21 14:25:46 2021 from 172.22.0.3
-bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
root@srv1:~#
```

In order for _salt-sproxy_ to have these credentials, let's put them into a Pillar file, referencing the `ssh` Proxy 
Module:

`/srv/salt/pillar/ssh.sls`

```yaml
proxy:
  proxytype: ssh
  host: {{ opts.id }}
  user: root
  priv: /etc/salt/ssh_key
  priv_passwd: APNIC2021
  ignore_host_keys: true
```

The Pillar Top file uses this file and assigns it to any `srv` device:

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
```

Similarly, in the Roster file, `/etc/salt/roster` it would suffice to have the following lines:

`/etc/salt/roster`

```yaml
srv1: {}
srv2: {}
srv3: {}
srv4: {}
```

With these, we can start running commands against the servers managed through SSH:

```bash
root@salt:~# salt-sproxy srv* test.ping
srv1:
    True
srv3:
    True
srv2:
    True
srv4:
    True
root@salt:~# salt-sproxy srv* grains.items

... snip ...
```

From here on, we can use _salt-sproxy_ to manage these servers, just like we would have through a regular Minion 
installed directly on them.

---
**End of Lab**

---
