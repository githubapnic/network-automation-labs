![](images/apnic_logo.png)
# LAB: Extending Salt In Your Own Environment

While Salt provides a significant number of native features and integrations with various tools, it cannot simply solve all the possible needs you might have. One obvious example is integrating Salt with internally developed tools that you have in your own environment; or just you want some very specific business logic that solves your requirements.

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

We have spoken previously about `file_roots`: this is the place where Salt is firstly looking for files (templates, State SLS, and so on). But this is also the place where it is looking for custom modules.

Extending a specific interface abides to same general rule: under your file system (for example, under one of the paths provided in `file_roots`), your provide the extension modules into a directory `_<module type>`, where module type can be `modules`, `proxy`, `grains`, `states`, `runners`, etc. For example, if you would like a new Execution Module, you would place it under the `_modules`, if you want new Grains, define a Python module under `_grains`, State Module under `_states`, Runners under `_runners`, and so on.

In all the following parts we will have the same `file_roots` we've had in all the previous labs so far:

```bash
grep file_roots -A 3 /etc/salt/master
```

<pre>
file_roots:
  base:
    - /srv/salt
    - /srv/salt/states
</pre>

The extension modules will therefore be physically located as follows:

- Execution Modules in `/srv/salt/_modules`
- Grains in `/srv/salt/_grains`
- Runners in `/srv/salt/_runners`
- Roster in `/srv/salt/_roster`
- Returner Modules in `/srv/salt/_returners`

## Part-3: Writing Runners

Unlike the majority of the Salt subsystems we have seen so far, which were strongly related to the (Proxy) Minions, Salt Runners are strongly tied to the Salt Master.

### Master side versus Minion side

Salt can be somewhat complicated to follow at first, as there are two perspectives: the Master side and the Minion side. 
What this actually means, is really just where the Python code is running - either on the machine of the Master, or the machine of the Salt Minion; these, in general are separate entities, and therefore this abstraction.

To be more specific, the following subsystems relate to the Minion side:

- Proxy Modules (implicitly, for Proxy Minions)
- Execution Modules (although, as we'll see later, they _can_ be run from the Master side too - but their designed for 
  Minion usage primarily)
- Grains Modules
- State Modules
- Beacons
- Engines
- Returners
- SDB
- Serializers
- Renderers
- Transport


And the following are designed for Master usage:

- Runners
- Pillars
- Returners
- NetAPI
- Beacons
- Engines
- SDB
- Tops
- Serializers
- Renderers
- Transport
- File servers
- Outputters
- ACL
- Auth

In fact, it turns out that most of the Salt subsystems relate to the Master. You may be surprised to see _Pillar_ listed in the above: we've previously have said that Pillar is data introduced into the system, and it is Minion-specific. Then how come this is coupled to the Master? The answer is simple: the Master is who computes the Pillar for each Minion -when a new Minion connects to the Master, during the Minion initiation process, after it has authenticated to the Master, it requests its Pillar data; the Master computes it and sends it over a secure channel. This is why, it's important to keep in mind two important details when it comes to Pillars: (1) Keep it lightweight, otherwise you'll put too much workload on the Master, and (2) This is the best place to store sensitive data such as passwords, tokens etc.

There are other subsystems that are specific to both Master and Minion, such as Returners, Serializers, Transport, Beacons or Engines, as they both make sense depending on the context they are used.

### Introduction to Salt Runners

A component that is tightly coupled to the Master is Runners. They behave just like Execution Modules, with the (big) difference that they run on the Master side. As a result, this has a number of implications:

- Runners don't have a `__pillar__` object, as the Master doesn't have Pillar associated with.
- Similarly, don't have a `__grains__` object, due to the same reason.
- The `__opts__` object, which was providing the _Minion_ configuration in the Execution Modules, in Runners provides the _Master_ configuration instead.
- In the exact same way, `__salt__` is no longer mapping the Execution functions loaded, rather the Runners loaded.

We will explore these in the next sections.

Another implication is the CLI syntax for executing Runners: as Runners are invoked on the Master side, we are not targeting other machines, and therefore the targeting parameter disappears:

<pre>
# salt-run &lt;runner.function&gt; [&lt;argument&gt;] [&lt;options&gt;]
</pre>

Notice that instead of the `salt` program, we're using `salt-run`, and there is no target (as the target is implicit -the very machine we're executing from). Just like the `salt` program, a Runner is invoked by referencing the Runner module named followed by a dot and the function name. Example:

```bash
salt-run test.arg
```

<pre>
root@salt:~# salt-run test.arg
args:
kwargs:
    ----------
</pre>

```bash
salt-run test.arg foo bar=baz
```

<pre>
root@salt:~# salt-run test.arg foo bar=baz
args:
    - foo
kwargs:
    ----------
    bar:
        baz
</pre>

In the above, the Runner invoked is `test.arg` (`test` is the Runner module, and `arg` is the function name), which only displays the arguments the function has been invoked with.

There are many useful Runners natively available in Salt. In fact, we've already seen one of them, a few modules ago, when watching the Salt Event Bus, by running the `salt-run state.event pretty=True` command. We now understand that by running that command, we've invoked the `state.event` Runner, which is executed on the Master, and therefore has access to seeing the entire event Master-Minion interaction, as well as external events.

Another handy Runner is `survey.hash`. This one executes a function (provided by the user), and then groups the Minions based on the output of that function. For example, if we'd like to group the devices based on the OS name, we'd use `survey.hash`, requesting to inspect the output of `grains.get os`:

```bash
salt-run survey.hash \* grains.get os
```

<pre>
root@salt:~# salt-run survey.hash \* grains.get os
|_
  ----------
  pool:
      - leaf1
      - leaf2
      - leaf3
      - leaf4
  result:
      ios
|_
  ----------
  pool:
      - spine1
      - spine2
      - spine3
      - spine4
  result:
      eos
|_
  ----------
  pool:
      - core1
      - core2
  result:
      iosxr
|_
  ----------
  pool:
      - router1
      - router2
  result:
      junos
</pre>

### Writing Runners

Writing Runners is simple, and testing them tends to be much simpler than Execution Modules, as everything happens on the local Master machine which generally means faster feedback loop.

As detailed at the beginning at this lab, our Runners are stored under the `/srv/salt/_runners` directory.

### Our first Runner

Just like the Execution Modules, Runners are free form, generally plain Python files with one or more functions. Let's start with an easy Runner that only returns the boolean value `True` when executed:

```bash
cat <<EOF > /srv/salt/_runners/example.py
def test():
    return True
EOF
```

Just like with any extensible component, we now need to synchronize this newly defined piece of code - again, with the difference that the code will be made available on the Master side, which means we'll use the `salt-run` command:

```bash
salt-run saltutil.sync_runners
```

<pre>
root@salt:~# salt-run saltutil.sync_runners
- runners.example
</pre>

With the `example` Runner now being available on the Master, we can execute:

```bash
salt-run example.test
```

<pre>
root@salt:~# salt-run example.test
True
</pre>

### Invoking other Runners

As mentioned in the previous paragraphs, we are able to invoke other Runners available. The methodology is very similar to the Execution Modules, by using the `__salt__` magic variable (which, as a reminder, on the Master it maps to the Runner functions, instead of the Execution functions):


### Invoking Execution Modules

While the Execution Modules are specific to the Minion, there are parts of code that may be useful on the Master side too, without having to duplicate the code. This works in most of the cases, however you need to be mindful of what the function invoked does, otherwise it may lead to unexpected behaviour.

To invoke Execution functions on the Master side, we can use the `salt.cmd` Runner, example:

```bash
salt-run salt.cmd test.ping
```

<pre>
root@salt:~# salt-run salt.cmd test.ping
True
</pre>

The call above executes the code of the simplest function `test.ping`, on the Master, then returns.

The code is being executed with the Master-specific environment. To verify this, check out the output of `test.get_opts` Execution Function when being run on the Master:

```bash
salt-run salt.cmd test.get_opts
```

<pre>
root@salt:~# salt-run salt.cmd test.get_opts
__cli:
    salt-run
__role:
    master
allow_minion_key_revoke:
    True

... snip ...
</pre>

`test.get_opts` basically just returns the `__opts__` object, which is nothing else than the entire configuration. Which configuration? That depends where the code is being executed: when `test.get_opts` is executed using the usual `salt` program, `__opts__` provides the Minion configuration, while on the Master, it provides the Master configuration:

```bash
salt router1 test.get_opts
```

<pre>
root@salt:~# salt router1 test.get_opts
router1:
    ----------
    __cli:
        salt-minion
    __role:
        minion
    acceptance_wait_time:
        10

... snip ...
</pre>

This is one important detail to always remember when calling Execution Functions on the Master side.

Other than that, another aspect to keep in mind is that any action takes place on the local Master too. For example, if we were to calculate the disk usage of a directory (in bytes), we'd get the following output:

```bash
salt-run salt.cmd file.diskusage /tmp/
```

<pre>
root@salt:~# salt-run salt.cmd file.diskusage /tmp/
20815
</pre>

In opposition, running this function on a Minion would, of course, return a different value, e.g.,

```bash
salt router1 file.diskusage /tmp/
```

<pre>
root@salt:~# salt router1 file.diskusage /tmp/
router1:
    1835
</pre>

We're equally able to invoke the custom Execution Modules defined earlier. For this, first step is to synchronize them on the Master side:

```bash
salt-run saltutil.sync_modules
```

<pre>
root@salt:~# salt-run saltutil.sync_modules
- modules.example
- modules.lab
- modules.show_arista
- modules.show_cisco
- modules.show_juniper
</pre>

With that, we can start executing:

```bash
salt-run salt.cmd example.true
```

<pre>
root@salt:~# salt-run salt.cmd example.true
True
</pre>

And also:

```bash
salt-run salt.cmd example.ping
```

<pre>
root@salt:~# salt-run salt.cmd example.ping
True
</pre>

```bash
salt-run salt.cmd example.ex_true
```

<pre>
root@salt:~# salt-run salt.cmd example.ex_true
True
</pre>

Which is "expected", as both `example.ping` and `example.ex_true` returned `True` previously when run on the Minions. 
But let's have a look again at the code:

```bash
grep -E -A 1 "ping\(|true\(" /srv/salt/_modules/example.py
```

<pre>
def ping():
    return __salt__['test.ping']()


def ex_true():
    return __salt__['example.true']()
</pre>

Here's something interesting: the code makes use of the `__salt__` variable to cross-call the `test.ping` and `example.true` functions, respectively. But we've said that `__salt__` maps to the Runners when executed on the Master, haven't we? Yes, but that is _inside_ the Runner functions; `salt.cmd` invokes the Execution Functions with the code they have been defined, only the _context_ is different (i.e., configuration options, Grains or Pillar data).

Let's also execute the `example.os_name` function:

```bash
salt-run salt.cmd example.os_name
```

<pre>
root@salt:~# salt-run salt.cmd example.os_name
Debian
</pre>

It didn't error or return empty, why?

Let's remember the code of this function:

```bash
grep -A 1 "os_name" /srv/salt/_modules/example.py
```

<pre>
def os_name():
    return __grains__['os']
</pre>

We've said previously that the Master doesn't have Grains associated with. And this statement remains true. But that doesn't mean it can't compute Grains on demand. In fact, when `salt.cmd` is being executed, it loads the Salt function context using local data.

In fact, you can even do this:

```bash
salt-run salt.cmd grains.items
```

<pre>
root@salt:~# salt-run salt.cmd grains.items
biosreleasedate:
    12/12/2017
biosversion:
    20171212
cpu_flags:
    - fpu
    - vme
    - de

... snip ...
</pre>

With this, executing `example.hello` becomes possible:

```bash
salt-run salt.cmd example.hello
```

<pre>
root@salt:~# salt-run salt.cmd example.hello
Hello stranger
</pre>

The output is _Hello stranger_ as the operating system is neither Junos, EOS, IOS or IOS-XR.

We have also mentioned that the Master doesn't have any Pillar data associated. It does however have access to seeing the Pillar of any Minion (as the Pillar data is compiled on the Master), using the `pillar.show_pillar` Runner:

```bash
salt-run pillar.show_pillar
```

<pre>
root@salt:~# salt-run pillar.show_pillar
devices:
    ----------
    router1:
        ----------
        role:
            router
    router2:
        ----------
        role:
            router
    core1:
        ----------
        role:
            core

... snip ...
</pre>

This returns the Pillar data common to every Minion connected to this Master. If we want something more specific, say check the Pillar data for `router1`, can use:

```bash
salt-run pillar.show_pillar router1
```

The latter provides the common data, as well as the NetBox data and the Proxy authentication specific to the `router1` Minion.

We can also call the `pillar.items` Execution Function, through the `salt.cmd` Runner:

```bash
salt-run salt.cmd pillar.items
```

<pre>
root@salt:~# salt-run salt.cmd pillar.items
[ERROR   ] Unable to pull NetBox data for "group00_master"
devices:
    ----------
    router1:
        ----------
        role:
            router
    router2:
        ----------
        role:
            router
    core1:
        ----------
        role:
            core
</pre>

This returns the common Pillar data, and would also attempt to pull the Pillar data for the `group00_master`. What is this `group00_master`. By default, Salt assigns an implicit ID for the Master, which is the machine hostname followed by `_master`. This can be verified by running:

```bash
salt-run salt.cmd config.get id
```

<pre>
root@salt:~# salt-run salt.cmd config.get id
group00_master
</pre>

Adding the following line to the master configuration file, `/etc/salt/master`:

```yaml
id: salt_master
```

This Master is not identified as `salt_master`, and the NetBox External Pillar would try to use this to pull the NetBox data:

```bash
salt-run salt.cmd pillar.items
```

<pre>
root@salt:~# salt-run salt.cmd pillar.items
[ERROR   ] Unable to pull NetBox data for "salt_master"
devices:
    ----------
    router1:
        ----------
        role:
            router
    router2:
        ----------
        role:
            router
    core1:
        ----------
        role:
            core
</pre>

As an exercise, define a new device in NetBox named `salt_master`. For this, you will also need to define:

- A new Role, for example _Salt Master_.
- A new Manufacturer, for example, _Docker_.
- A new Device Type, for example, _Debian_.
- A new Platform, for example _Debian 9_, referencing the newly created _Docker_ Manufacturer.
- Optionally, you may also define:
  * _Docker_ Cluster Type.
  * _Docker Compose_ Cluster Groups.
  * _Lab_ Cluster, of type _Docker_ and group _Docker Compose_.

And, finally, the `salt_master` device referencing these newly defined details.

Once the device is added, running `root@salt:/# salt-run salt.cmd pillar.items` we'll see NetBox data being populated.

--
**End of Lab**

---
