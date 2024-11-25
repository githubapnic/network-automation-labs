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

## Part-5: Writing Roster Modules

We have used Roster modules when working with Salt SProxy. Rosters are useful when managing devices _without_ running 
(Proxy) Minions for them - that is, the code is executed on demand whenever we make the call. When having (Proxy) 
Minions running, Salt knows what devices it has under its supervision; in opposition, when we don't have (Proxy) Minions 
running, Salt needs to be told what devices it needs to manage, i.e., and inventory of the device. In Salt, this is 
called Roster, an interface that provides the list of devices and their properties.

So far we've seen two Salt SProxy native Rosters at work:

1. `file`, when the list of devices has been provided statically, into the `/etc/salt/roster` SLS file.
2. `netbox`, when the list of devices has been retrieved from NetBox, via HTTP requests to the API.

Before diving into effectively writing our own Roster module, let's do some preparation work.

As mentioned in the previous lab, Salt SProxy is smart enough and it will allow you to execute on both devices you have 
and those devices you don't have running (Proxy) Minions for. This is done through an option named `use_existing_proxy` 
that is configured as `true` in the Master configuration file. In this lab we still have Proxy Minions running; for 
clarify and avoid any overlaps, we will want SProxy to use only its own set of devices (i.e., without the running Proxy 
Minions). For this all we have to do is disable the `use_existing_proxy` feature:

`/etc/salt/master`

```yaml
use_existing_proxy: false
```

Now, let's check the list of devices salt-sproxy is aware of right now:

```bash
root@salt:~# salt-sproxy \* --preview
- router1
- router2
- core1
- core2
- spine1
- spine2
- spine3
- spine4
- leaf1
- leaf2
- leaf3
- leaf4
- srv1
- srv2
- srv3
- srv4
```

All of these are defined in the `/etc/salt/roster` file as configured via the `roster: file` and `roster_file: 
/etc/salt/roster` configuration options.

The `/etc/salt/roster` file is interpreted as SLS. Open it and remember its Jinja + YAML structure. As always in Salt, 
these files are very powerful and they allow you to make them as dynamic as you want. For example, remove its contents 
entirely:
```bash
rm /etc/salt/roster && touch /etc/salt/roster
```

```bash
root@salt:~# rm /etc/salt/roster && touch /etc/salt/roster
root@salt:~#
```

Now, as there are no devices defined in the Roster file, salt-sproxy returns:
```bash
salt-sproxy \* --preview
```

```bash
root@salt:~# salt-sproxy \* --preview
No devices matched your target. Please review your tgt / tgt_type arguments, or the Roster data source
```

### Roster SLS

We are going to use the HTTP API in order to populate the Roster SLS file, instead of managing it manually, exploiting 
the powers of SLS rendering.

Using the `http.query` Runner, execute the following request:
```bash
salt-run http.query http://http_api:8888
```

```bash
root@salt:~# salt-run http.query http://http_api:8888
body:
    {"devices": {"router1": {"role": "router"}, "router2": {"role": "router"}, "core1": {"role": "core"}, "core2": {"role": "core"}, "spine1": {"role": "spine"}, "spine2": {"role": "spine"}, "spine3": {"role": "spine"}, "spine4": {"role": "spine"}, "leaf1": {"role": "leaf"}, "leaf2": {"role": "leaf"}, "leaf3": {"role": "leaf"}, "leaf4": {"role": "leaf"}}}
```

`http://http_api:8888` is the HTTP API we've used in a previous lab, for the External Pillars.

The data returned is JSON format, so we're able to un-serialize (decode) it:
```bash
salt-run http.query http://http_api:8888 decode=True --out=raw
```

```bash
root@salt:~# salt-run http.query http://http_api:8888 decode=True --out=raw
{'body': '{"devices": {"router1": {"role": "router"}, "router2": {"role": "router"}, "core1": {"role": "core"}, "core2": {"role": "core"}, "spine1": {"role": "spine"}, "spine2": {"role": "spine"}, "spine3": {"role": "spine"}, "spine4": {"role": "spine"}, "leaf1": {"role": "leaf"}, "leaf2": {"role": "leaf"}, "leaf3": {"role": "leaf"}, "leaf4": {"role": "leaf"}}}', 'dict': {'devices': {'router1': {'role': 'router'}, 'router2': {'role': 'router'}, 'core1': {'role': 'core'}, 'core2': {'role': 'core'}, 'spine1': {'role': 'spine'}, 'spine2': {'role': 'spine'}, 'spine3': {'role': 'spine'}, 'spine4': {'role': 'spine'}, 'leaf1': {'role': 'leaf'}, 'leaf2': {'role': 'leaf'}, 'leaf3': {'role': 'leaf'}, 'leaf4': {'role': 'leaf'}}}}
```

The decoded / un-serialized object is under the `dict` key. In other words, just by invoking the `http.query` Runner and 
looking under the `dict` key, we're able to gather the list of devices from the HTTP API, in a format that Python can 
understand, and we are thus able to use inside the SLS file:

`/etc/salt/roster`

```sls
{{ __salt__.http.query('http://http_api:8888', decode=True)['dict']['devices'] }}
```

This is the entire Roster we need: it executes the HTTP request to `http://http_api:8888`, deserializes the data, which 
we can find under the `dict` and `devices` keys respectively.

To confirm, we can execute:
```bash
salt-sproxy \* --preview
```

```bash
root@salt:~# salt-sproxy \* --preview
- router1
- router2
- core1
- core2
- spine1
- spine2
- spine3
- spine4
- leaf1
- leaf2
- leaf3
- leaf4
```

Or, in debug mode:
```bash
salt-sproxy \* --preview -l debug
```

```bash
root@salt:~# salt-sproxy core* --preview -l debug
[DEBUG   ] Reading configuration from /etc/salt/master
[DEBUG   ] Using cached minion ID from /etc/salt/minion_id: npnog10-vm00

... snip ...

[DEBUG   ] Requesting URL http://http_api:8888 using GET method
[DEBUG   ] Using backend: tornado
[DEBUG   ] Response Status Code: 200
[PROFILE ] Time (in seconds) to render '/etc/salt/roster' using 'jinja' renderer: 0.02762770652770996
[DEBUG   ] Rendered data from file: /etc/salt/roster:
{'router1': {'role': 'router'}, 'router2': {'role': 'router'}, 'core1': {'role': 'core'}, 'core2': {'role': 'core'}, 'spine1': {'role': 'spine'}, 'spine2': {'role': 'spine'}, 'spine3': {'role': 'spine'}, 'spine4': {'role': 'spine'}, 'leaf1': {'role': 'leaf'}, 'leaf2': {'role': 'leaf'}, 'leaf3': {'role': 'leaf'}, 'leaf4': {'role': 'leaf'}}

[DEBUG   ] Results of YAML rendering:
OrderedDict([('router1', OrderedDict([('role', 'router')])), ('router2', OrderedDict([('role', 'router')])), ('core1', OrderedDict([('role', 'core')])), ('core2', OrderedDict([('role', 'core')])), ('spine1', OrderedDict([('role', 'spine')])), ('spine2', OrderedDict([('role', 'spine')])), ('spine3', OrderedDict([('role', 'spine')])), ('spine4', OrderedDict([('role', 'spine')])), ('leaf1', OrderedDict([('role', 'leaf')])), ('leaf2', OrderedDict([('role', 'leaf')])), ('leaf3', OrderedDict([('role', 'leaf')])), ('leaf4', OrderedDict([('role', 'leaf')]))])
[PROFILE ] Time (in seconds) to render '/etc/salt/roster' using 'yaml' renderer: 0.003397226333618164
[DEBUG   ] Glob matching on dict_items([('router1', {'minion_opts': OrderedDict([('role', 'router')])}), ('router2', {'minion_opts': OrderedDict([('role', 'router')])}), ('core1', {'minion_opts': OrderedDict([('role', 'core')])}), ('core2', {'minion_opts': OrderedDict([('role', 'core')])}), ('spine1', {'minion_opts': OrderedDict([('role', 'spine')])}), ('spine2', {'minion_opts': OrderedDict([('role', 'spine')])}), ('spine3', {'minion_opts': OrderedDict([('role', 'spine')])}), ('spine4', {'minion_opts': OrderedDict([('role', 'spine')])}), ('leaf1', {'minion_opts': OrderedDict([('role', 'leaf')])}), ('leaf2', {'minion_opts': OrderedDict([('role', 'leaf')])}), ('leaf3', {'minion_opts': OrderedDict([('role', 'leaf')])}), ('leaf4', {'minion_opts': OrderedDict([('role', 'leaf')])})]) ? core*
[DEBUG   ] The target expression "core*" (glob) matched the following:
[DEBUG   ] ['core1', 'core2']

... snip ...

- core1
- core2
```

The debug logs confirm that the HTTP request is taking place, and the `/etc/salt/roster` is rendered as we would expect.

### Our first Roster module

Using the Roster SLS is a powerful way to create dynamic inventories, however it may not always be the best solution -
or sometimes this may not be a possibility at all.

Around the same example, let's craft a Roster module that queries the same HTTP endpoint http://http_api:8888, and 
returns the list of devices.

Just line with the Returner modules, Rosters are free form Python modules, with only one minor constraint: the main 
function has a specific signature: the name is `targets()` and it must accept one positional argument `tgt` and 
a keyword argument `tgt_type` which specify the targeting expression and the targeting expression type, respectively.

Another element of importance that we will need is invoking the `http.query` Runner (as we've done above). This is 
accessible through a special variable `__runner__` which gives access to all the Runners Salt is aware of (just like the 
`__salt__` dunder in the Runner functions).

At first, the implementation we would be tempted to propose would be:

`/srv/salt/_roster/example.py`

```python
def targets(tgt, tgt_type='glob', **kwargs):
    devices = __runner__['http.query']('http://http_api:8888', decode=True)['dict']['devices']
    return devices
```

As the Rosters are Master-specifics, we would need to use `salt-run` in order to synchronize the new code:
```bash
salt-run saltutil.sync_roster
```

```bash
root@salt:~# salt-run saltutil.sync_roster
- roster.example
```

As the new `example` Roster is now available, let's ensure _salt-sproxy_ is using it:

`/etc/salt/master`

```yaml
roster: example
```

After pointing _salt-sproxy_ to use the new `example` Roster, the following request would work:
```bash
salt-sproxy \* --preview
```

```bash
root@salt:~# salt-sproxy \* --preview
- router1
- router2
- core1
- core2
- spine1
- spine2
- spine3
- spine4
- leaf1
- leaf2
- leaf3
- leaf4
```

In debug mode (re-running the previous command with `-l debug`), we observe the logging is slightly different than 
previously when using the Roster SLS:

```
[DEBUG   ] Computing the target using the example Roster
[DEBUG   ] LazyLoaded example.targets
[DEBUG   ] Requesting URL http://http_api:8888 using GET method
[DEBUG   ] Using backend: tornado
[DEBUG   ] Response Status Code: 200
[DEBUG   ] The target expression "*" (glob) matched the following:
[DEBUG   ] ['router1', 'router2', 'core1', 'core2', 'spine1', 'spine2', 'spine3', 'spine4', 'leaf1', 'leaf2', 'leaf3', 'leaf4']
```

We're still seeing the log corresponding to the HTTP request to http://http_api:8888, as we would expect, and besides 
that, the logs also say that the `example.targets` Roster function is being executed. This would suggest the `example` 
Roster works OK so far.

Let's try and run a different targeting expression, for example:
```bash
salt-sproxy core* --preview
```

```bash
root@salt:~# salt-sproxy core* --preview
- router1
- router2
- core1
- core2
- spine1
- spine2
- spine3
- spine4
- leaf1
- leaf2
- leaf3
- leaf4
```

That is not what we would expect, so we have to correct the `example` Roster. To understand what it happens, we can 
re-run in debug mode. We notice the following two lines:

```
[DEBUG   ] The target expression "core*" (glob) matched the following:
[DEBUG   ] ['router1', 'router2', 'core1', 'core2', 'spine1', 'spine2', 'spine3', 'spine4', 'leaf1', 'leaf2', 'leaf3', 'leaf4']
```

This seems to be the culprit: no matter what targeting expression we request, the Roster always returns the entire list 
of devices. To correct this, we can implement our own mechanisms to compute the targeting; or, easier, make use of 
_salt-sproxy_ features. The Roster becomes:

`/srv/salt/_roster/example.py`

```python
import salt_sproxy._roster

def targets(tgt, tgt_type='glob', **kwargs):
    devices = __runner__['http.query']('http://http_api:8888', decode=True)['dict']['devices']
    engine = salt_sproxy._roster.TGT_FUN[tgt_type]
    return engine(devices, tgt, opts=__opts__)
```

_salt-sproxy_ offers a set of features for implementing Rosters, and just before returning, we can invoke one of the 
integrated target matching engines (for each target type). Resync, and then the targeting works as expected:
```bash
salt-run saltutil.sync_roster
```

```bash
root@salt:~# salt-run saltutil.sync_roster
- roster.example
```
```bash
salt-sproxy core* --preview
```
```bash
root@salt:~# salt-sproxy core* --preview
- core1
- core2
```

As one can easily notice, writing Roster modules is equally simple and straight forward. Depending on what the business 
requirements are, we can implement solutions for building the inventory for the devices managed through Salt SProxy 
or even Salt SSH.

--
**End of Lab**

---
