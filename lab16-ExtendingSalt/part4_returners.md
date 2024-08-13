![](images/apnic_logo.png)
# LAB: Extending Salt In Your Own Environment

While Salt provides a significant number of native features and integrations with various tools, it cannot simply solve 
all the possible needs you might have. One obvious example is integrating Salt with internally developed tools that you 
have in your own environment; or just you want some very specific business logic that solves your requirements.

For this, Salt is capable to offer the entire functionality set for you to easily extend it in your own environment.

Salt has a high cardinal of module types, of which we've seen just a few so far:

- Execution Modules
- Proxy Modules
- Grains
- States
- Pillars (i.e., External Pillar modules)
- Runners (we'll visit Runners in _Part-3_)
- Returners
- Rosters
- Beacons
- Engines
- NetAPI
- SDB (i.e., Salt Database)
- Tops (i.e., modules for building dynamic Top Files)
- Transport (i.e., modules for the Minion-Master communication, by default ZeroMQ)
- Serializers
- Renderers
- File servers (i.e., where Salt would be retrieving the files from - e.g., Git, S3, Azure, SVN, etc.)
- Outputters (in what format to display the data, using the `--out` CLI option)
- ACL
- Auth

Every single interface is pluggable in your own environment.

We have spoken previously about `file_roots`: this is the place where Salt is firstly looking for files (templates, 
State SLS, and so on). But this is also the place where it is looking for custom modules.

Extending a specific interface abides to same general rule: under your file system (for example, under one of the paths 
provided in `file_roots`), your provide the extension modules into a directory `_<module type>`, where module type can 
be `modules`, `proxy`, `grains`, `states`, `runners`, etc. For example, if you would like a new Execution Module, you 
would place it under the `_modules`, if you want new Grains, define a Python module under `_grains`, State Module under 
`_states`, Runners under `_runners`, and so on.

In all the following parts we will have the same `file_roots` we've had in all the previous labs so far:

`/etc/salt/master`

```yaml
file_roots:
  base:
    - /srv/salt
    - /srv/salt/states
```

The extension modules will therefore be physically located as follows:

- Execution Modules in `/srv/salt/_modules`
- Grains in `/srv/salt/_grains`
- Runners in `/srv/salt/_runners`
- Roster in `/srv/salt/_roster`
- Returner Modules in `/srv/salt/_returners`

## Part-4: Writing Returners

We have spoken so far about modules that are either Minion or Master specific (again, with the distinction that the 
Master is capable to access and run Minion code as well, when requested to). In this part we'll visit a special Salt 
subsystem named _Returners_ which is neither Master or Minion specific, as it can be used on both sides, depending on 
the use case or design.

### Introduction to Returners

Returners are a Salt subcomponent, which, as the name would suggest, forward the job returns to third party systems or 
services, outside of Salt.

In order to have Salt return the data to an external service, simply append the `--return` option followed by the 
Returner name:

```bash
salt <taget> <module.function> [<arguments>] [<options>] --return <returner>
```

Example:

```bash
salt router1 test.ping --return redis
```

### Returners usage example: Redis

One of the easiest to use Returner module is _Redis_, which provides the interface to storing Salt returns into a Redis 
service. Redis is an in-memory data structure store, used as a database, cache or message broker. In simpler terms, you 
an think of Redis as a database engine which is easy to work with and stores the data in memory.

For this lab, we have a Redis instance running locally, available at the hostname `redis`, port `6379` (the default 
Redis port).

Salt provides natively various types of modules to facilitate the interaction with Redis, such as Returners or Execution 
Modules.

For the Redis Returner and Execution Module, we have the following options pre-defined in the Proxy configuration file:

`/etc/salt/proxy`

```yaml
redis.db: '1'
redis.host: 'redis'
redis.port: 6379
```

Redis can operate with up to 16 databases, from 0 to 15. For this scenario, we are using database `1`. The Salt modules 
for Redis additionally require the `redis` Python package which is pre-installed everywhere in the lab.

With all these requirements being met, we can go ahead and run:
```bash
salt router1 test.ping --return redis
```

```bash
root@salt:~# salt router1 test.ping --return redis
router1:
    True
```

From a command line perspective, nothing has changed, but we expect that the return has been forwarded to the Redis 
server. Let's see what keys we have stored in Redis:
```bash
salt router1 redis.keys
```

```bash
root@salt:~# salt router1 redis.keys
router1:
    - ret:20210215180740059045
    - minions
    - router1:test.ping
```

There are three keys. In order to see the content of each key, we also need to know the data type. The key which has the 
job execution result is the key beginning with `ret:`:
```bash
salt router1 redis.key_type ret:20210215180740059045
```

```bash
root@salt:~# salt router1 redis.key_type ret:20210215180740059045
router1:
    hash
```

The `redis.key_type` function tells us that `ret:20210215180740059045` is a _hash_. Therefore, we need to execute the 
appropriate operation for hash types. This is `redis.hgetall` (i.e., _hash get all_) to return all the field of this 
hash:
```bash
salt router1 redis.hgetall ret:20210215180740059045
```

```bash
root@salt:/# salt router1 redis.hgetall ret:20210215180740059045
router1:
    ----------
    router1:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215180740059045", "fun": "test.ping", "fun_args": [], "id": "router1"}
```

This shows the job execution has been correctly stored, as we would expect.

Knowing that the return is stored in Redis under a key named `ret:<JID>`, let's display the JID when executing the 
following command:
```bash
salt \* test.ping --show-jid --return redis
```

```bash
root@salt:~# salt \* test.ping --show-jid --return redis
jid: 20210215184323092800
leaf2:
    True
spine2:
    True
spine1:
    True
leaf4:
    True
spine4:
    True
leaf3:
    True
core1:
    True
core2:
    True
leaf1:
    True
spine3:
    True
router1:
    True
router2:
    True
```

The JID is `20210215184323092800` (on your machine it will certainly be a different JID, so replace it in the below 
command), let's check what we have in Redis under the `ret:20210215184323092800`:

```bash
salt router1 redis.hgetall ret:20210215184323092800
```

```bash
root@salt:~# salt router1 redis.hgetall ret:20210215184323092800
router1:
    ----------
    core1:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "core1"}
    core2:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "core2"}
    leaf1:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "leaf1"}
    leaf2:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "leaf2"}
    leaf3:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "leaf3"}
    leaf4:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "leaf4"}
    router1:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "router1"}
    router2:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "router2"}
    spine1:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "spine1"}
    spine2:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "spine2"}
    spine3:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "spine3"}
    spine4:
        {"success": true, "return": true, "retcode": 0, "jid": "20210215184323092800", "fun": "test.ping", "fun_args": [], "id": "spine4"}
```

Each execution return for the `20210215184323092800` job is stored under under this key, for each Minion that has 
matched the target (in this case, all).

### Our first custom Returner module

For example, let's write a simple Returner that saves a configuration backup whenever the execution of the 
`net.load_config` or `net.load_template` function is detected.

As we're accustomed already, extension modules are just Python files stored under a specific location from where Salt 
loads them. This location is, for the lab, is `/srv/salt/_returners`.

Returner modules are being invoked by passing the `--return <returner>` option, which is referring the Returner name 
only; whenever we had worked with custom modules previously, the pattern was `<module>.<function>`. Now, the function is 
implicit, it's always `returner()`, hence the Returners come with this minor constraint: the module can define as many 
Python functions as required, but Salt will always invoke `returner()`. That means, for the Returner module to be 
valid, the `returner()` function must be implemented.

Inside the Returner modules, we're able to use the `__salt__` variable which provides access to invoking any Execution 
Function.

With these, the Returner module for backing up configuration would be as simple as:

`/srv/salt/_returners/example.py`

```python
def returner(ret):
    if ret['fun'] in ('net.load_config', 'net.load_template'):
        __salt__['net.save_config'](source='running', path='/tmp/bkup')
```

The Returner consists solely on the `returner()` function. One important aspect to notice is that the `returner()` 
function accepts one argument, and that is the event return. As a reminder, open the event bus in a separate terminal 
window (by running `salt-run state.event pretty=True`) and execute `salt router1 net.load_config text='set system ntp 
server 10.0.0.1' test=True` to check the structure of the return event:

```
salt/job/20210216120855776443/ret/router1	{
    "_stamp": "2021-02-16T12:08:56.312004",
    "cmd": "_return",
    "fun": "net.load_config",
    "fun_args": [
        {
            "test": true,
            "text": "set system ntp server 10.0.0.1"
        }
    ],
    "id": "router1",
    "jid": "20210216120855776443",
    "retcode": 0,
    "return": {
        "already_configured": false,
        "comment": "Configuration discarded.",
        "diff": "[edit system]\n+   ntp {\n+       server 10.0.0.1;\n+   }",
        "loaded_config": "",
        "result": true
    },
    "success": true
}
```

The `ret` object that is passed in to the `returner()` function is the exact event body / data. In other words, the 
`ret` object is a Python dictionary with usual event return keys, `fun`, `fun_args`, `id`, `jid`, etc.

Inside the `returner()` function we can inspect these elements if we want to ensure we won't process every single 
return. More specifically, if we want to only backup the configuration when executing the configuration management 
commands `net.load_config` and `net.load_template`, we can check the value of the `fun` field - hence the `if ret['fun'] 
in ('net.load_config', 'net.load_template'):` statement in the first line of the function body.
When condition is met, the `net.save_config` is being invoked, to save the running configuration to `/tmp/bkup`. Let's 
see this in action: first step is synchronising the new Returner:

```bash
root@salt:~# salt router1 saltutil.sync_returners
router1:
    - returners.example
```

Before running, let's check the contents of the `/tmp` directory:
```bash
salt router1 cmd.run 'ls -la /tmp'
```

```bash
root@salt:~# salt router1 cmd.run 'ls -la /tmp'
router1:
    total 8
    drwxrwxrwt 1 root root 4096 Feb 16 12:19 .
    drwxr-xr-x 1 root root 4096 Feb 16 12:07 ..
```

Executing without `--return example`, there's no change:
```bash
salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True
```

```bash
root@salt:~# salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True
router1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        [edit system]
        +   ntp {
        +       server 10.0.0.1;
        +   }
    loaded_config:
    result:
        True
root@salt:~# salt router1 cmd.run 'ls -la /tmp'
router1:
    total 8
    drwxrwxrwt 1 root root 4096 Feb 16 12:19 .
    drwxr-xr-x 1 root root 4096 Feb 16 12:07 ..
```

And now, finally, let's run with `--return example`:

```bash
salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True --return example
```

```bash
root@salt:~# salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True --return example
router1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        [edit system]
        +   ntp {
        +       server 10.0.0.1;
        +   }
    loaded_config:
    result:
        True
```

Checking the contents of the `/tmp` directory, we see the file is there and can check its contents:
```bash
salt router1 cmd.run 'ls -la /tmp'
```
```bash
root@salt:~# salt router1 cmd.run 'ls -la /tmp'
router1:
    total 12
    drwxrwxrwt 1 root root 4096 Feb 16 12:26 .
    drwxr-xr-x 1 root root 4096 Feb 16 12:07 ..
    -rw-r--r-- 1 root root 1384 Feb 16 12:26 bkup
root@salt:~#
root@salt:~#
```
```bash
salt router1 file.read /tmp/bkup
```

```bash
root@salt:~# salt router1 file.read /tmp/bkup
router1:
    ## Last commit: 2021-02-16 12:20:58 UTC by apnic
    version 17.2R1.13;
    system {
        host-name router1;
        root-authentication {
```

With this, we can see how easily it is to craft a custom Returner. Another key point to remember is that not everything 
must be sent to the storage destination, as the returner can be modeled to only save the data we are looking for. You 
can look at the Returner as an alternative to the Reactor system when you want to kick off job execution in response to 
return events.

--
**End of Lab**

---
