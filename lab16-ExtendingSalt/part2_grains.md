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

## Part-2: Writing Grains Modules

Grains modules, just like the Execution Modules are nothing else than Python modules (files) that has a very specific 
scope. Grains modules typically have one or more Python functions returning Grain data. The Python functions are 
free-form and allow you to do any necessary computation; the data returned is going to be registered as Grains.

There are no constraints, however there are two (logical) implications:

- The functions _must_ return data.
- The data returned has to be serializable.
- As each Grain must have a name and a value, the data returned by the Grain functions is typically a dictionary (with 
  as many nesting levels as required).

Grains modules, in order to be loaded correctly, require one additional piece of information, at the top of the Python 
module:

`/srv/salt/_grains/example.py`

```python
__proxyenabled__ = ['*']
```

The `__proxyenabled__` variable tells Salt to load this Grains module _only_ when the Proxy Minions is one of the listed 
ones; for simplicity, we can use `*` which means that the Grains module will be loaded for any Proxy Minion type.
This can be very handy to designate what Grains of modules to be loaded, depending on the properties and the features 
available for each Proxy Minion type.

### The simplest Grain function

As when crafting Execution Modules, the simplest way to get started is with a function that only returns a single value. 
For example, a flag:

`/srv/salt/_grains/example.py`

```python
def lab():
    return {'lab': True}
```

All this function does is returns a dictionary with a single key-value pair (the key is `lab`, and the value is `True`). 
Returning this structure, Salt will register the value `lab` as a Grain, and `True` as its value. To check this, we need 
to synchronize the new code on the Minions, by running the `saltutil.sync_grains` function:

```bash
root@salt:~# salt \* saltutil.sync_grains
leaf4:
    - grains.example
leaf3:
    - grains.example
spine2:
    - grains.example
router2:
    - grains.example
leaf2:
    - grains.example
router1:
    - grains.example
spine4:
    - grains.example
spine1:
    - grains.example
core1:
    - grains.example
spine3:
    - grains.example
core2:
    - grains.example
leaf1:
    - grains.example
```

And now, let's check the value of the `lab` Grain:

```bash
root@salt:~# salt \* grains.get lab
spine2:
    True
leaf1:
    True
leaf4:
    True
leaf3:
    True
spine1:
    True
router2:
    True
core2:
    True
leaf2:
    True
spine3:
    True
spine4:
    True
core1:
    True
router1:
    True
```

This Grain can now be used for targeting and many other use-cases:

```bash
root@salt:~# salt -G lab:True --preview
- router1
- spine1
- spine3
- spine4
- leaf2
- spine2
- leaf4
- core1
- core2
- leaf3
- router2
- leaf1
```

### Same function, multiple Grains

A common misconception around the Grains modules is that every function can only return one single value / Grain. This 
is false, every key in the returned dictionary will be registered as a Grain. Let's consider the following example:

`/srv/salt/_grains/example.py`

```python
import random

def extra_info():
    return {'type': 'training', 'location': ['virtual', 'everywhere'], 'random': random.randint(1,12)}
```

This function returns three key-value pairs; notice that the values can have any datatype we want - as long as it's 
a Python native type:

- `type` is a string.
- `location` is a list of strings.
- `random` is an integer. For the sake of diversity, it is just a random number, without any particular significance.

Synchronize the changes (`salt \* saltutil.sync_grains`) then we can see the new Grains:

```bash
root@salt:~# salt router1 grains.get type
router1:
    training
root@salt:~# salt spine1 grains.get location
spine1:
    - virtual
    - everywhere
root@salt:~# salt core2 grains.get random
core2:
    4
```

As always, we can use these Grains for targeting:

```bash
root@salt:~# salt -G random:4 --preview
- spine2
- core2
root@salt:~# salt -G location:virtual --preview
- router1
- leaf3
- router2
- spine3
- core1
- spine1
- core2
- spine4
- leaf1
- spine2
- leaf4
- leaf2
```

### Multiple level Grains

Grains not always have to be 1:1 key-value pairs; the value of a specific Grain can also be defined as a more complex 
object:

`/srv/salt/_grains/example.py`

```python
def multi():
    return {
        'level1': {
            'level2': {
                'level3': 'some-value'
            }
        }
    }
```

This can be seen in (after sync):

```bash
root@salt:~# salt router1 grains.get level1
router1:
    ----------
    level2:
        ----------
        level3:
            some-value
```

Targeting on multiple levels, can be done by separating the levels by colon (`:`):

```bash
root@salt:~# salt -G level1:level2:level3:some-value --preview
- spine3
- core1
- router1
- leaf2
- leaf4
- spine4
- spine2
- router2
- spine1
- leaf3
- leaf1
- core2
```

### Dynamic Grains functions

So far, all the Grains functions from the previous examples were returning statical information. But the most important 
side of writing Grains functions is that they can return dynamic data instead.

As a prime example, let's define a function that returns whether the Proxy Minion is running under a Docker container. 
For this all we have to do is verify whether the file `/.dockerenv` exists:

`/srv/salt/_grains/example.py`

```
import os

def docker():
    return {
        'docker': os.path.exists('/.dockerenv')
    }
```

This simple function returns the `docker` Grain, with the value from the Python standard library `os.path.exists` which 
return `True` if the named file exists and `False` otherwise.

After synchronizing the Grain module, we can find out that all our Proxy Minions are running in Docker containers:

```bash
root@salt:~# salt \* grains.get docker
spine4:
    True
spine2:
    True
leaf1:
    True
spine1:
    True
core2:
    True
leaf3:
    True
core1:
    True
spine3:
    True
router2:
    True
router1:
    True
leaf4:
    True
leaf2:
    True
```

### Accessing configuration options in the Grains

Just as with the Execution Modules, we are able to access the Minion configuration data, through the `__opts__` 
variable.

One of the most interesting options is `id` which is nothing else than the Minion ID, for example:

```bash
root@salt:~# salt router1 config.get id
router1:
    router1
```

We can access this value by looking up under the `id` index in `__opts__`:

`/srv/salt/_grains/example.py`

```python
def opts():
    return {'device_name': __opts__['id']}
```

The new Grain `device_name` returns the Minion ID value for each:

```bash
root@salt:~# salt \* grains.get device_name
leaf2:
    leaf2
spine3:
    spine3
spine4:
    spine4
spine1:
    spine1
spine2:
    spine2
router1:
    router1
core1:
    core1
core2:
    core2
leaf3:
    leaf3
leaf1:
    leaf1
leaf4:
    leaf4
router2:
    router2
```

While this Grain is not particularly useful for CLI targeting, it can be very handy in various other contexts such as 
templates, SLS files etc.

In production, you won't need to define this Grain like this, as there is the `id` Grain you can use - but now you know 
what it runs under the hood:

```bash
root@salt:~# salt \* grains.get id
router1:
    router1
spine1:
    spine1
spine3:
    spine3

...
```

### Accessing Pillar data in the Grains

In the exact same way as with the Execution Modules, Pillar data associated with a specific Minion can be accessed from the `__pillar__` magic variable.

From the previous labs, remember in the Pillar, among other data, we have a list of devices retrieved via the `http_json` External Pillar:

```bash
salt router1 pillar.get devices
```

<pre>
root@salt:~# salt router1 pillar.get devices
router1:
    ----------
    core1:
        ----------
        role:
            core
    core2:
        ----------
        role:
            core
    leaf1:
        ----------
        role:
            leaf
...
</pre>

For programability reasons, it may be easier to visualise this data as a Python object instead:

```bash
salt router1 pillar.get devices --out=raw
```

<pre>
root@salt:~# salt router1 pillar.get devices --out=raw
{'router1': {'router1': {'role': 'router'}, 'router2': {'role': 'router'}, 'core1': {'role': 'core'}, 'core2': {'role': 'core'}, 'spine1': {'role': 'spine'}, 'spine2': {'role': 'spine'}, 'spine3': {'role': 'spine'}, 'spine4': {'role': 'spine'}, 'leaf1': {'role': 'leaf'}, 'leaf2': {'role': 'leaf'}, 'leaf3': {'role': 'leaf'}, 'leaf4': {'role': 'leaf'}}}
</pre>

As the command above shows, underneath the `devices` Pillar key, there's a dictionary whose keys are the device names. 
From a Python function we can just return what's under that device-name key and register as Grains:

Back up the file first
```bash
cp /srv/salt/_grains/example.py ~/example.py.grains.bak
```

```bash
cat <<EOF > /srv/salt/_grains/example.py

def pillar():
    device_id = __opts__['id']
    return __pillar__.get('devices', {}).get(device_id)
EOF
```

The function above does just this, and after a sync, we can check that the `role` field is not present as a Grain:

```bash
salt \* saltutil.sync_modules
```

```bash
salt \* grains.get role
```

<pre>
root@salt:~# salt \* grains.get role
spine1:
    spine
spine2:
    spine
router1:
    router
leaf1:
    leaf
leaf2:
    leaf
spine3:
    spine
spine4:
    spine
router2:
    router
core1:
    core
core2:
    core
leaf4:
    leaf
leaf3:
    leaf
</pre>

In this lab, we have covered the Grain modules essentials. Without diving into more advanced pure Python structured, we can now start crafting our own Grains, as the business logic requires.

--
**End of Lab**

---
