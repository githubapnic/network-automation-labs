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

## Part-1: Writing Execution Modules

Execution Modules are nothing else than Python modules that are loaded by Salt and made available in various contexts 
such as CLI, Jinja templates (and therefore SLS files), other Execution Modules, State modules, Reactors, and pretty 
much almost every other Salt component. They are perhaps the most flexible type of Salt modules, having access to many 
data entities such as Pillar, Grains, or configuration options.

Every Salt module consists on one or more Python functions. When Salt loads the modules, it registers the function under 
the name `<module>.<function>`.

### The simplest Execution Module

One of the easiest native functions is `test.true` which only returns `True` - nothing else. To understand how this 
function is defined, let's craft our own, say `example.first`. That is, a function named `first()` into a module named 
`example.py`. As previously mentioned, the module is physically located under the `/srv/salt/_modules` directory:

`/srv/salt/_modules/example.py`

```bash
cat /srv/salt/_modules/example.py
```

```python
def first():
    return True
```

This is the entire content required for the `example.py` module.

Now, in order to have the Minions aware of a change in this file, we execute:

```bash
salt \* saltutil.sync_modules
```

```bash
root@salt:~# salt \* saltutil.sync_modules
spine4:
    - modules.example
leaf3:
    - modules.example
leaf2:
    - modules.example
spine1:
    - modules.example
leaf1:
    - modules.example
spine3:
    - modules.example
spine2:
    - modules.example
core2:
    - modules.example
core1:
    - modules.example
router2:
    - modules.example
router1:
    - modules.example
leaf4:
    - modules.example
```

By running `saltutil.sync_modules` we tell to every Minion that the `example.py` file has been updated and should 
re-fetch the new code from the Master. The output `modules.example` confirms that the Minion has retrieved the new code, 
and it now has it available to be executed:

```bash
salt router1 example.first
```

```bash
root@salt:~# salt router1 example.first
router1:
    True
```

Say we wanted to be able to execute `example.true` instead of `example.first`. We would be tempted to define the 
function as following:

```python
def true():
    return True
```

The code would work, but this is not a good idea, as the name `true` is a reserved keyword in Python; by defining this 
function we would override that reserved keyword and may result in unexpected behaviour. A better approach is to define 
the function with a different name, such as `test_()` and tell Salt to re-map it to `example.first`. This can be done 
through a special instruction defined at the top of the file, `__func_alias__`, which maps the actual function name to 
the name we want Salt to register it:

`/srv/salt/_modules/example.py`

```python
__func_alias__ = {
    'true_': 'true',
}


def true_():
    return True
```




Sync then we can run `example.true`:

```bash
salt router1 saltutil.sync_modules
salt router1 example.true
```

```bash
root@salt:~# salt router1 saltutil.sync_modules
router1:
    - modules.example
root@salt:~# salt router1 example.true
router1:
    True
```

Without the `__func_alias__` definition map, the function would still be available, but we'd have to run it as 
`example.true_`:

```bash
root@salt:~# salt router1 example.true_
router1:
    True
```

There is another element of detail to keep in mind when defining Salt functions: **functions starting with underscore
are never loaded**. Say we wanted to define our function as `_true()` instead of `true_()`, like this:

`/srv/salt/_modules/example.py`

```python
def _true():
    return True
```

If defining `true_()` we'd call the function as `example.true_`, we would probably expect that, if we call the function 
`_true()`, we'll access it as `example._true`; but this is false, as the function is not loaded because it starts with 
an underscore character:

```bash
root@salt:~# salt router1 example._true
router1:
    'example._true' is not available.
```

Salt returns an error and non-zero return code. As a general rule, by convention, functions starting with an underscore 
are considered "private" functions and therefore not loaded. They can however be used as helpers to be called by other 
functions, for example:

`/srv/salt/_modules/example.py`

```python
def _help():
    return True


def first():
    return _help()
```

The function `first()` only invokes the helper function named `_help()`. In Salt, `_help()` won't be made available, but 
`first()` will:

```bash
salt router1 saltutil.sync_modules

salt router1 example._help

salt router1 example.first
```

```bash
root@salt:~# salt router1 saltutil.sync_modules
router1:
    - modules.example
root@salt:~# salt router1 example._help
router1:
    'example._help' is not available.
ERROR: Minions returned with non-zero exit code
root@salt:~# salt router1 example.first
router1:
    True
```

### Accessing Grains data

Grains data associated with a specific Minion can be accessed through the `__grains__` special variable. When a modules 
is loaded, Salt injects a number of "magic" variables into the context of each function. `__grains__` is one of them -
we'll see others in the next paragraphs.

Into the same `example.py` module let's define a simple function which only returns the value of the `__grains__` 
variable:

`/srv/salt/_modules/example.py`

```python
def grains():
    return __grains__
```

Sync then call the new `example.grains` function:

```bash
salt router1 example.grains
```

```bash
root@salt:~# salt router1 saltutil.sync_modules
router1:
    - modules.example
root@salt:~# salt router1 example.grains
router1:
    ----------
    cpuarch:
        x86_64
    cwd:
        /
    dns:
        ----------
        domain:
        ip4_nameservers:
            - 127.0.0.11
        ip6_nameservers:
        nameservers:
            - 127.0.0.11
        options:
            - ndots:0
        search:
        sortlist:
    fqdns:
    gpus:
    host:
        router1

...
... snip ...
...
```

Executing the same using the `--out=raw` outputter, we can notice that `__grains__` is nothing else than a Python 
dictionary, where the key is the Grain name, and the value is the Grain value - example: the value of the `host` Grain 
is `router1` for `router1`, etc. That said, in code, if we want to lookup for a specific value, we'd only need to do 
a simple Python dictionary lookup; let's define a Salt function `example.os_name` that returns the value of the `os` 
Grain:

`/srv/salt/_modules/example.py`

```python
def os_name():
    return __grains__['os']
```

Sync then run:

```bash
salt \* example.os_name
```

```bash
root@salt:~# salt \* example.os_name
router2:
    junos
leaf2:
    ios
spine1:
    eos
leaf1:
    ios
core2:
    iosxr
router1:
    junos
spine4:
    eos
leaf3:
    ios
spine3:
    eos
core1:
    iosxr
spine2:
    eos
leaf4:
    ios
```

The function returns the OS name for every individual device. Using Grains is a great way to build business logic for 
a given function, depending on what platform it is run against. For example, the following function `hello()`:

`/srv/salt/_modules/example.py`

```python
def hello():
    if __grains__['os'] == 'junos':
        return 'Hello Juniper'
    elif __grains__['os'] == 'iosxr':
        return 'Hello Cisco IOS-XR'
    elif __grains__['os'] == 'ios':
        return 'Hello classic IOS'
    elif __grains__['os'] == 'eos':
        return 'Hello Arista'
    return 'Hello stranger'
```

Depending on the `os` Grain, the function returns a different hello message:

```bash
salt \* example.hello
```

```bash
root@salt:~# salt \* example.hello
spine4:
    Hello Arista
leaf3:
    Hello classic IOS
spine1:
    Hello Arista
leaf1:
    Hello classic IOS
router1:
    Hello Juniper
leaf4:
    Hello classic IOS
core2:
    Hello Cisco IOS-XR
spine2:
    Hello Arista
router2:
    Hello Juniper
core1:
    Hello Cisco IOS-XR
leaf2:
    Hello classic IOS
spine3:
    Hello Arista
```

If we were to run on a different platform, the code wouldn't fail and would return `Hello stranger` instead.

In the same way we can use the Grains inside the Execution modules to model the behaviour depending on many other 
factors, such as chassis model, OS version, device role, etc. - the possibilities are unlimited, especially when you 
thing that you can provide additional Grains in your own environment.

### Accessing Pillar data 

In the same way we access Grain data, we can also access Pillar data associated with a certain Minion. This is done, as 
you may expect now, using the `__pillar__` magic variable injected into the function context:

`/srv/salt/_modules/example.py`

```python
def pillar():
    return __pillar__
```

Sync and execute:

```
salt router1 saltutil.sync_modules
```


```bash
root@salt:~# salt router1 saltutil.sync_modules
router1:
    - modules.example
root@salt:~# salt router1 example.pillar
router1:
    ----------
    devices:
        ----------
        core1:
            ----------
            role:
                core

... snip ...
```

The data structure largely depends on what you put into the Pillar, so it's always a good idea to consult it using the 
`--out=raw` outputter to see how Python interprets it.

The data structure again abides to the same rules, and we can return specific data from the Pillar:

`/srv/salt/_modules/example.py`

```python
def location():
    return 'I am in Rack {rack}, at {address}'.format(
        rack=__pillar__['netbox']['rack']['name'],
        address=__pillar__['netbox']['site']['physical_address']
    )
```

The `location()` function returns a text message providing the rack name and the address of the device:

After sync, we can now run:

```bash
salt \* example.location
```

```bash
root@salt:~# salt \* example.location
spine4:
    I am in Rack R8, at 6 Cordelia Street,
    South Brisbane,
    QLD 4101,
    Australia
spine2:
    I am in Rack R6, at 6 Cordelia Street,
    South Brisbane,
    QLD 4101,
    Australia
leaf1:
    I am in Rack R9, at 6 Cordelia Street,
    South Brisbane,
    QLD 4101,
    Australia
leaf2:
    I am in Rack R9, at 6 Cordelia Street,
    South Brisbane,
    QLD 4101,
    Australia

... snip ...
```

Similar to the Grains, we can use Pillar data to model business logic into the Execution modules based on different 
values we may want.

### Accessing configuration options

Along the same lines with `__grains__` and `__pillar__`, inside the execution modules we can also check the value of the 
configuration options, using the `__opts__` variable:

`/srv/salt/_modules/example.py`

```python
def opts():
    return __opts__
```

This method is not the most flexible in the context of Proxy Minions, as the configuration options are only loaded on 
process startup - so if you want to change the value of a specific configuration, you need to restart the Proxy Minion.
There is a more convenient way by using the `config.get` Execution function which merges the configuration options with 
the Pillar data. We will see in the next paragraph how we are able to call `config.get` and other Salt functions from 
a custom function.

### Invoking Salt functions

As you may have guessed, this is done through another special variable, which is `__salt__`. This variable is a plain 
dictionary whose keys are all the Salt functions loaded, and the values point to the memory address of the named 
function. This is simpler than it sounds, let's have an example:

`/srv/salt/_modules/example.py`

```python
def ping():
    return __salt__['test.ping']()
```

Sync and execute:

```bash
salt router1 example.ping
```

```bash
root@salt:~# salt router1 saltutil.sync_modules
router1:
    - modules.example
root@salt:~# salt router1 example.ping
router1:
    True
```

**Important**: Remember to always use the open-close braces `(` and `)` to properly invoke the function – `__salt__` is
a mapping of all the available functions Salt is aware of, each element pointing to the memory address where the
function is available for execution. Therefore, it is important to have the braces even though you may not pass any
arguments to the function.

In the same way we can invoke native Salt functions, such as `test.ping`, as well as functions defined into our own 
environment, for example `example.true` defined earlier:

`/srv/salt/_modules/example.py`

```python
def ex_true():
    return __salt__['example.true']()
```

```bash
salt router1 example.ex_true
```

```bash
root@salt:~# salt router1 example.ex_true
router1:
    True
```

### Invoking low-level API calls

In _Lab 13_ we have explored several functions that we can use - from the command line - to invoke low-level API calls:

- Arista, over eAPI, via the pyEAPI Python library.
- Junos, over NETCONF, through the existing junos. module.
- A variety of platforms, using screen-scraping over either SSH or Telnet (i.e., issue CLI commands and return the text
  output, as you’d normally have on your terminal when executing the command(s) manually).

As a brief summary, these are some of the most important functions we can use, per platform:

- Junos:
  * `napalm.junos_rpc`: execute RPC calls and return the output as Python object.
  * `napalm.junos_cli`: execute CLI commands and return the output as either text or Python object.
  * `napalm.junos_commit`: for more flexible options when committing the configuration on Junos, that NAPALM doesn’t
    currently have, e.g., commit confirmed, synchronise, force sync, etc.
- Arista:
  * `napalm.pyeapi_run_commands`: execute a list of CLI / show style commands and return the output as text or Python
    object.
  * `napalm.pyeapi_config`: configure the Arista switch with the specified commands.
- Any platform supported by Netmiko (including the ones above):
  * `napalm.netmiko_commands`: send one or more CLI commands to be executed on the device, and return the output as
    plain text.
  * `napalm.netmiko_config`: Load one or more configuration commands on the network device. The commands can equally be
    loaded from a local or remote path, and passed through Salt’s template rendering pipeline (by default using Jinja
    as the template rendering engine).

As any other Salt function, we are able to invoke these from a custom module and then process the output as needed.

#### Junos

For example, let's craft a function that executes an RPC request to a Juniper device to retrieve the OS version. 

The RPC call in this case can be found by running `show version | display xml rpc` on the device CLI:

```xml
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

Through Salt, we can execute this RPC call using the `napalm.junos_rpc` function:

```bash
salt router1 napalm.junos_rpc get-software-information --out=raw
```

```bash
root@salt:~# salt router1 napalm.junos_rpc get-software-information --out=raw
{'router1': {'comment': '', 'result': True, 'out': {'software-information': {'host-name': 'router1', 'product-model': 'vmx', 'product-name': 'vmx', 'junos-version': '17.2R1.13', 'package-information': []}}}}
```

The OS version is `17.2R1.13` and it is nested under a few levels of keys: `out`, `software-information`, and, finally, 
`junos-version`. The `router1` key is only relevant on the command line, in order to identify the device returning the 
output.

That said, we can define the following function, `junos_version()`:

`/srv/salt/_modules/example.py`

```python
def junos_version():
    ret = __salt__['napalm.junos_rpc']('get-software-information')
    return ret['out']['software-information']['junos-version']
```

In the first line of the function, we invoke the `napalm.junos_rpc`, using the `__salt__` variable, passing in the name 
of the RPC call, `get-software-information` - and save the result into a variable named `ret`. From that variable, we 
can look up under under the `out`, `software-information`, and `junos-version` keys, as mentioned above, to return the
Junos version:

```bash
root@salt:~# salt router* saltutil.sync_modules
router2:
    - modules.example
router1:
    - modules.example
root@salt:~# salt router* example.junos_version
router2:
    23.2R1.14
router1:
    23.2R1.14
```

#### Arista

For Arista switches, we can approach it in a similar way - using the `napalm.pyeapi_run_commands` function:

```bash
salt spine1 napalm.pyeapi_run_commands 'show version' --out=raw
```

```bash
root@salt:~# salt spine1 napalm.pyeapi_run_commands 'show version' --out=raw

{'spine1': [{'mfgName': 'Arista', 'modelName': 'cEOSLab', 'hardwareRevision': '', 'serialNumber': 'E588A1C410C66B07870AE0E5AA6A742E', 'systemMacAddress': '00:1c:73:09:53:a7', 'hwMacAddress': '00:00:00:00:00:00', 'configMacAddress': '00:00:00:00:00:00', 'version': '4.30.3M-33434233.4303M (engineering build)', 'architecture': 'x86_64', 'internalVersion': '4.30.3M-33434233.4303M', 'internalBuildId': 'e3c0ed3e-117f-4e32-9902-c7316ef78ca2', 'imageFormatVersion': '1.0', 'imageOptimization': 'None', 'cEosToolsVersion': '(unknown)', 'kernelVersion': '5.15.0-1065-gcp', 'bootupTimestamp': 1723422234.700414, 'uptime': 16231.578438520432, 'memTotal': 65928940, 'memFree': 24767472, 'isIntlVersion': False}]}
```

The EOS version is under key named `version`, with a catch: the keys are nested under a list; as a reminder, to 
`napalm.pyeapi_run_commands` we can provide a list of commands to be executed, and therefore the output is a list of 
outputs for each command. That said, the output of `show version` is indexed at position 0 of this output. In code, this 
would look like:

`/srv/salt/_modules/example.py`

```python
def eos_version():
    ret = __salt__['napalm.pyeapi_run_commands']('show version')
    return ret[0]['version']
```

The logic is even simpler - store the output of the code-equivalent of `napalm.pyeapi_run_commands 'show version'` into 
the `ret` variable, then do a lookup under the index 0, key `version` to return the EOS version. As always, synchronize 
and run:

```bash
root@salt:~# salt spine* saltutil.sync_modules
spine4:
    - modules.example
spine3:
    - modules.example
spine2:
    - modules.example
spine1:
    - modules.example
root@salt:~# salt spine* example.eos_version
spine1:
    4.30.3M-33434233.4303M (engineering build)
spine3:
    4.30.3M-33434233.4303M (engineering build)
spine4:
    4.30.3M-33434233.4303M (engineering build)
spine2:
    4.30.3M-33434233.4303M (engineering build)
```

#### Cisco IOS-XR

While Juniper and Arista offer features and APIs for easily interact with the network device, Cisco plaforms, in general 
don't provide use with enough features, or they aren't simple enough to extend on top of them. This is why, we will have 
to fall back to using Netmiko to execute show-like commands, and TextFSM templates to parse that text output. In _Lab 7_ 
we have analysed the structure and the usage of TextFSM templates to extract information from `show version` commands on 
IOS-XR devices. One of the command we've used then (on the command line) was `textfsm.extract`; in a custom Salt module, 
we can use that function, again, to process the raw output from `show version`.

Let's have a look at the `napalm.netmiko_commands` that we can use to gather the text output from `show version`:

```bash
salt core1 napalm.netmiko_commands 'show version'
```

```bash
root@salt:~# salt core1 napalm.netmiko_commands 'show version'
core1:
    -
      Mon Aug 12 04:59:36.436 UTC

      Cisco IOS XR Software, Version 6.0.1[Default]
      Copyright (c) 2016 by Cisco Systems, Inc.

      ROM: GRUB, Version 1.99(0), DEV RELEASE

      core1 uptime is 4 hours, 38 minutes
      System image file is "bootflash:disk0/xrvr-os-mbi-6.0.1/mbixrvr-rp.vm"

      cisco IOS XRv Series (Pentium Celeron Stepping 3) processor with 3145215K bytes of memory.
      Pentium Celeron Stepping 3 processor at 2810MHz, Revision 2.174
      IOS XRv Chassis

      7 GigabitEthernet
      1 Management Ethernet
      97070k bytes of non-volatile configuration memory.
      866M bytes of hard disk.
      2321392k bytes of disk0: (Sector size 512 bytes).
... snip ...
```

The `iosxr_version()` function would look like:

```python
def iosxr_version():
    sh_ver = __salt__['napalm.netmiko_commands']('show version')[0]
    fsm = __salt__['textfsm.extract']('salt://textfsm/iosxr_show_version.fsm', raw_text=sh_ver)
    return fsm['out'][0]['version']
```

The output from `show version` executed through `napalm.netmiko_commands` is stored into `sh_ver` then passed in to 
`textfsm.extract` which using the `salt://textfsm/iosxr_show_version.fsm` TextFSM template extracts the information into 
the `fsm` variable, from where we can return the version from under the `version` key:

```bash
salt core1 example.iosxr_version
```

```bash
root@salt:~# salt core1 saltutil.sync_modules
core1:
    - modules.example
root@salt:~# salt core1 example.iosxr_version
core1:
    6.0.1
```

#### Cisco IOS

For IOS, the implementation would be very similar to the one proposed above for IOS-XR; the main difference would be the 
TextFSM template, as the output of `show version` is different. In that case, we'd probably be tempted to implement the 
`ios_version()` function as follows:

```python
def ios_version():
    sh_ver = __salt__['napalm.netmiko_commands']('show version')[0]
    fsm = __salt__['textfsm.extract']('salt://textfsm/ios_show_version.fsm', raw_text=sh_ver)
    return fsm['out'][0]['version']
```

Notice that this would be the exact same piece of code as above, except that we're referencing the 
`ios_show_version.fsm` here instead of `iosxr_show_version.fsm`. In programming there's a general recommendation to 
avoid redundant code (KISS - Keep It Simple and Stupid). With that in mind, let's remember another detail from _Lab 7_: 
using the _TextFSM index file_, we managed to execute the same command across both Cisco platforms, and the TextFSM 
index detects the platforms and designates which template to use in order to extract the data:

```bash
salt -G vendor:Cisco net.cli "show version" textfsm_parse=True textfsm_path=salt://textfsm/
```

```bash
root@salt:~# salt -G vendor:Cisco net.cli "show version" textfsm_parse=True textfsm_path=salt://textfsm/
leaf2:
    ----------
    comment:
    out:
        ----------
        show version:
            |_
              ----------
              config_register:
                  0x2102
              hardware:
                  - CSR1000V
              hostname:
                  leaf2
              mac:
              reload_reason:
                  Reload Command
              rommon:
                  IOS-XE
              running_image:
                  packages.conf
              serial:
                  - 9P1Q2T5XYOE
              uptime:
                  4 hours, 41 minutes
              version:
                  16.12.4
    result:
        True
core2:
    ----------
    comment:
    out:
        ----------
        show version:
            |_
              ----------
              build_host:
              hardware:
                  IOS XRv Series
              location:
              uptime:
                  4 hours, 39 minutes
              version:
                  6.0.1
    result:
        True

...
... snip ...
...
```

In that case, let's look into using the `net.cli` function instead:

```python
def ios_version():
    cli = __salt__['net.cli']('show version', textfsm_parse=True, textfsm_path='salt://textfsm/')
    return cli['out']['show version'][0]['version']
```

The `net.cli "show version" textfsm_parse=True textfsm_path=salt://textfsm/` call on the command line is transformed 
into the Python `__salt__['net.cli']('show version', textfsm_parse=True, textfsm_path='salt://textfsm/')` instruction, 
with the output stored into the variable named `cli`; the version is then extracted from the `out`, `show version` and 
`version` keys - as this is the return structure from `net.cli`.

With this change, we can now call `ios_version()` across any Cisco platform:

```bash
salt -G vendor:Cisco example.ios_version
```

```bash
root@salt:~# salt -G vendor:Cisco example.ios_version
core2:
    6.0.0
core1:
    6.0.0
leaf1:
    15.5(2)S
leaf4:
    15.5(2)S
leaf2:
    15.5(2)S
leaf3:
    15.5(2)S
```

### Abstracted, cross-platform, functions

With the previous example, when using the TextFSM index file across multiple platforms, we've made a step forward 
towards cross-platform implementations. But while TextFSM can help in some certain situations, like the one above, it is 
not always the answer: not only that parsing bits of text is fragile but TextFSM templates are not always trivial to 
implement and debug - while when an API is available it is much more straight forward to use it. As a rule of thumb, use 
the API whenever possible, and TextFSM templates only as a last-resort.

That said, we want a Salt function, say `example.version` which returns the OS version for whatever platform (so we 
don't have to run separate functions for specific platforms). More specifically, this is what we want to achieve:

```bash
salt \* example.version
```

```bash
root@salt:~# salt \* example.version
spine4:
    4.30.3M-33434233.4303M (engineering build)
spine3:
    4.30.3M-33434233.4303M (engineering build)
spine1:
    4.30.3M-33434233.4303M (engineering build)
spine2:
    4.30.3M-33434233.4303M (engineering build)
leaf2:
    16.12.4
core2:
    6.0.1
core1:
    6.0.1
leaf4:
    16.12.4
leaf3:
    16.12.4
leaf1:
    16.12.4
router2:
    23.2R1.14
router1:
    23.2R1.14
```

For this, we only have to remember the _Accessing Grains data_ section and using the `__grains__` variable to model 
business logic depending on various factors such as operating system or vendor:

```python
def version():
    if __grains__['os'] == 'junos':
        return junos_version()
    elif __grains__['os'] == 'eos':
        return eos_version()
    elif __grains__['vendor'] == 'Cisco':
        return ios_version()
```

In this we can notice the configuration bits we have prepared above: if the function is executed on a Juniper device, 
execute `junos_version()`, on Arista invoke `eos_version()`, and, finally, on any Cisco platform call `ios_version()`.
Synchronize and execute:

```bash
root@salt:~# salt \* saltutil.sync_modules
router2:
    - modules.example
spine2:
    - modules.example
core1:
    - modules.example
spine4:
    - modules.example
router1:
    - modules.example
leaf3:
    - modules.example
spine3:
    - modules.example
leaf2:
    - modules.example
spine1:
    - modules.example
core2:
    - modules.example
leaf4:
    - modules.example
leaf1:
    - modules.example
root@salt:~# salt \* example.version
spine4:
    4.18.1F
spine2:
    4.18.1F
spine3:
    4.18.1F
core2:
    6.0.0
core1:
    6.0.0
spine1:
    4.18.1F
leaf1:
    15.5(2)S
leaf2:
    15.5(2)S
leaf4:
    15.5(2)S
leaf3:
    15.5(2)S
router2:
    17.2R1.13
router1:
    17.2R1.13
```

This is how you can easily implement your own cross-platform Salt function, that abstract away the differences between 
vendors and platforms, and return you the data you are looking for, regardless of their internal representation.

### Virtual modules

Our `version()` function above is simple enough to understand and read, as it's still divided into separate functions. 
A more condensed version of it, without the helper functions `junos_version`, `ios_version`, ..., would be the 
following:

```python
def version():
    if __grains__['os'] == 'junos':
        ret = __salt__['napalm.junos_rpc']('get-software-information')
        return ret['out']['software-information']['junos-version']
    elif __grains__['os'] == 'eos':
        ret = __salt__['napalm.pyeapi_run_commands']('show version')
        return ret[0]['version']
    elif __grains__['vendor'] == 'Cisco':
        cli = __salt__['net.cli']('show version', textfsm_parse=True, textfsm_path='salt://textfsm/')
        return cli['out']['show version'][0]['version']
```

A function like this is still readable and easy to follow - but when we would like to build something more complex with 
more business logic added, it can become rather cumbersome.

But Salt can help with this too: instead of having a single, complicated, function, with everything needed for all the 
platforms, we can instead break it apart into multiple physical files - but all registered under the same name space. 
Salt, unlike most of the automation frameworks, has this unique feature to allow you to register a module under 
a virtual name. So far, our module was named `example.py` and made available on the CLI (and other Salt subsystems) as 
`example.` module; this is the implicit behaviour. If, say we wanted the module to be available under a different name, 
for example, `virtual_example`, we would require the following special function defined at the top of the module:

```python
def __virtual__():
    return 'virtual_example'
```

Just by having this, our module would continue to be named `example.py` (as a file), but Salt will know it as 
`virtual_example`, and therefore our functions will be named `virtual_example.test`, `virtual_example.version` and so 
on.

This feature is more powerful that just renaming a module: we can have multiple files pointing to the same virtual 
module name and loaded based on particular behaviour. For example, we can have these 3 files:

- `show_juniper.py`, for Juniper-related show commands.
- `show_arista.py`, for Arista-related show commands.
- `show_cisco.py` for Cisco-related show commands.

They all will be registered under the virtual name `show`, so when we run a specific function against a Juniper device 
it will be loaded from `show_juniper.py`, when running against Arista, from `show_arista.py`, and so on.

This can be done easily through the `__virtual__` function, which decides whether to load the module depending on Grains 
or other properties:

`/srv/salt/_modules/show_juniper.py`

```python
def __virtual__():
    if __grains__['os'] == 'junos':
        return 'show'
    return False, 'This works only on Junos'
```

The `__virtual__()` function in `/srv/salt/_modules/show_juniper.py` tells Salt "_load the code from this file, **only** 
when the device targeted is a Juniper_".

Similarly, the other two files:

`/srv/salt/_modules/show_arista.py`

```python
def __virtual__():
    if __grains__['os'] == 'eos':
        return 'show'
    return False, 'This works only on Arista EOS'
```

And:

`/srv/salt/_modules/show_cisco.py`

```python
def __virtual__():
    if __grains__['vendor'] == 'Cisco':
        return 'show'
    return False, 'This works only on Cisco platforms'
```

The `__virtual__()` functions in each of these files would ensure the module is loaded under the virtual name `show`, 
depending on the platform targeted.

Inside each file we now defined the `version()` function with the appropriate code, depending on the targeted platform:

- Juniper:

`/srv/salt/_modules/show_juniper.py`

```python
def __virtual__():
    if __grains__['os'] == 'junos':
        return 'show'
    return False, 'This works only on Junos'

def version():
    ret = __salt__['napalm.junos_rpc']('get-software-information')
    return ret['out']['software-information']['junos-version']
```

- Arista:

`/srv/salt/_modules/show_arista.py`

```python
def __virtual__():
    if __grains__['os'] == 'eos':
        return 'show'
    return False, 'This works only on Arista EOS'


def version():
    ret = __salt__['napalm.pyeapi_run_commands']('show version')
    return ret[0]['version']
```

- Cisco:

`/srv/salt/_modules/show_cisco.py`

```python
def __virtual__():
    if __grains__['vendor'] == 'Cisco':
        return 'show'
    return False, 'This works only on Cisco platforms'


def version():
    cli = __salt__['net.cli']('show version', textfsm_parse=True, textfsm_path='salt://textfsm/')
    return cli['out']['show version'][0]['version']
```

The implementation of the `version()` function in each file depends on the platform. But what it stays is that the 
files are all registered under the `show` virtual name, and they all have the function `version()` defined.

That said, we can go ahead and synchronize:

```bash
root@salt:~# salt \* saltutil.sync_modules
spine2:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
leaf3:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
spine1:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
spine4:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
spine3:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
leaf1:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
core2:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
leaf2:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
leaf4:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
router1:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
router2:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
core1:
    - modules.show_arista
    - modules.show_cisco
    - modules.show_juniper
```

Notice that in the output, all three files are loaded. But when executing, only the appropriate one will be invoked:

```bash
salt \* show.version
```

```bash
root@salt:~# salt \* show.version
router2:
    17.2R1.13
router1:
    17.2R1.13
spine3:
    4.18.1F
spine2:
    4.18.1F
spine1:
    4.18.1F
spine4:
    4.18.1F
leaf1:
    15.5(2)S
core2:
    6.0.0
leaf2:
    15.5(2)S
leaf3:
    15.5(2)S
leaf4:
    15.5(2)S
core1:
    6.0.0
```

With this methodology we can abstract away operations across multiple platforms, and keep the code decoupled which may 
be simpler in various contexts; this is also a win in terms of self-explanatory documentation (i.e., the code for
Juniper goes into `/srv/salt/_modules/show_juniper.py`, for Arista into `/srv/salt/_modules/show_arista.py`, and so on).

--
**End of Lab**

---
