![](images/apnic_logo.png)
# LAB: Introduction to configuration management using Salt

As we will be showcasing different mechanisms, depending on the underlying library used (i.e., NAPALM, Netmiko or junos-enzc), the Proxy Minions are not already started, and we will need to start them up as needed - after updating the Pillar data to point to the appropriate Proxy Module.

Confirm the Pillar Top File is configured as follows:

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

This structure ensures that the routers use the **junos.sls** file, the cores are provided with the Pillar data from **iosxr.sls** and so on. In the next sections we will be looking at each of these files individually and update them with the desired contents.

## Part-1: Configuration management using NAPALM

### Preparing the environment

Firstly, let's go through each of the `junos.sls`, `iosxr.sls`, `eos.sls` and `ios.sls` Pillar file to ensure they are pointing to the NAPALM Proxy Module.

Confirm the Pillar SLS Files have **napalm** configured as the proxytype:

```bash
grep -in napalm /srv/salt/pillar/*.sls
```

<pre>
root@salt:~# grep -in napalm /srv/salt/pillar/*.sls
/srv/salt/pillar/eos.sls:2:  proxytype: napalm
/srv/salt/pillar/ios.sls:2:  proxytype: napalm
/srv/salt/pillar/iosxr.sls:2:  proxytype: napalm
/srv/salt/pillar/junos.sls:2:  proxytype: napalm
</pre>

To view the contents of junos.sls:

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

Here, we find the structure and the credentials as configured in _Lab #3 (Configuring Proxy Minions)_. The `proxytype` points to `napalm` as expected, and the `driver` is `junos` as the SLS file is used to managing Junos devices.

**Important**
One particular detail to note is the `host: {{ opts.id }}`. To understand this, remember that SLS is, by default, a combination of Jinja + YAML, i.e., the file is rendered as a Jinja template, then the result is loaded as a YAML data structure. In this case, `{{ opts.id }}` is just a piece of a Jinja template, which means: replace here with the value of the `id` field from the `opts` object. But what is the `opts` object? It simply points to the configuration options of the Proxy Minions (see _Lab #1 (Configuring Salt)_), and `id` is the Minion ID.

In short, the declaration `host: {{ opts.id }}` is interpreted as `host: router1` for the `router1` Minion, `host: router2` for the `router2` Minion, and so on. This is a good and flexible way to have one file being used across multiple devices.

Take a moment and go through the rest of the Pillar files.

Once we've verified the contents of the files, let's start one Proxy Minion for each role. Initially, in debug mode to ensure the Proxy Minion is able to start up correctly:

```bash
salt-proxy -l debug --proxyid router1
```

If everything goes normally, we can then SIGKILL (Ctrl-C) the process, and start it in daemon mode:

```bash
salt-proxy --proxyid router1 -d
```

**Note**: There may be a python error, but the command completes succesfully

<pre>
  root@salt:~# salt-proxy --proxyid router1 -d
/usr/local/lib/python3.6/site-packages/requests/__init__.py:91: RequestsDependencyWarning: urllib3 (1.26.18) or chardet (3.0.4) doesn't match a supported version!
  RequestsDependencyWarning)
</pre>

Add **2> /dev/null** to the end of the commands to suppress the python error. For example:

```bash
salt-proxy --proxyid router1 -d 2> /dev/null
```

Continue the same operation for `core1`, `spine1`, and `leaf1`. 

```bash
salt-proxy --proxyid core1 -d
salt-proxy --proxyid leaf1 -d
salt-proxy --proxyid spine1 -d
```

Verify that the Proxy Minions respond correctly, by running, for example:

```bash
salt -L router1,core1,spine1,leaf1 test.ping
```

<pre>
root@salt:~# salt -L router1,core1,spine1,leaf1 test.ping
leaf1:
    True
router1:
    True
spine1:
    True
core1:
    True
</pre>

### The NAPALM configuration management functions

NAPALM offers two functions for CLI usage:

- `net.load_config`: load configuration as text or from a specific file.
- `net.load_template`: render a template then load the resulting configuration on the device.

#### `net.load_config`

**net.load_config** has the following (optional) arguments:

- **filename**: The path to the file to load the configuration from. This can be an absolute path to a physical file, or a remote file accessed via `salt://`, `http(s)://`, `ftp://`, `s3://`, `swift://`, etc.
- **text**: In-line configuration passed in via the command line.
- **test**: Whether to commit the config, or only execute a dry-run. Default value: `False` (would commit the config). When passed in as `True`, it would discard the config, but still returning the config diff.
- **commit**: Whether to commit the config. Default value: `True`. The difference between `commit` and `tests` is that when `commit` is passed in as `False`, it doesn't discard the loaded configuration changes but simply not commit them.
- **replace**: Boolean value, defaulting to `False`, whether we want to fully replace the running configuration with the newly loaded one. By default, `net.load_config` performs a _merge_ into the running config.
- **debug**: Another boolean flag which can be used to check what config we are loading.

Let's start by deploying some simple configuration changes:

```bash
salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True debug=True
```

<pre>
root@salt:~# salt router1 net.load_config text='set system ntp server 10.0.0.1' test=True debug=True
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
        set system ntp server 10.0.0.1
    result:
        True
</pre>

The return has a number of elements:

- **already_configured**: A boolean that states whether there are any configuration changes.
- **comment**: A human-understandable message (can be a detailed explanation in case there's an error).
- **diff**: The configuration diff (i.e., between the running config and the config with the newly loaded changes).
- **loaded_config**: Provides the exact configuration being loaded, _only_ when the `debug=True` flag is passed.
- **result**: A boolean telling whether the execution succeeded.

Let's now put the configuration above into a file, and have `net.load_config` use that:

```bash
cat /srv/salt/static/junos
```

<pre>
root@salt:~# cat /srv/salt/static/junos
set system ntp server 10.0.0.1
</pre>

```bash
salt router1 net.load_config filename=/srv/salt/static/junos test=True
```

<pre>
root@salt:~# salt router1 net.load_config filename=/srv/salt/static/junos test=True
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
</pre>

But there's a catch with this method: the filename **/srv/salt/static/junos** has been passed in using the absolute path; the only reason it works now is that the Proxy Minion runs on the same machine as the Master. If the Proxy Minion runs elsewhere - as it's usually the case in practice - this would fail. For this reasoning, it's best to place the files under one of the Salt file system paths, and use the **salt://** URI to have the Minions access the file uniformly:

```bash
salt router1 net.load_config filename=salt://static/junos test=True debug=True
```

<pre>
root@salt:~# salt router1 net.load_config filename=salt://static/junos test=True debug=True
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
        set system ntp server 10.0.0.1
    result:
        True
</pre>

This is where the `debug=True` flag comes in handy too.

#### `net.load_template`

`net.load_template` is very similar as in CLI arguments and return structure to `net.load_config`, with the difference that `net.load_template` renders a template and then it loads the resulting configuration.

The template engine is _Jinja_ by default, with the possibility to choose between other languages, such as Cheetah, Genshi, Mako, WemPy, or even pure Python. This can be toggled using the `template_engine` argument.

The usage is simple, instead of passing in the location of a static file, you will provide the location of the template. Let's consider the following simple template:

```bash
cat /srv/salt/templates/iosxr.jinja
```
<pre>
root@salt:# cat /srv/salt/templates/iosxr.jinja
  
hostname {{ opts.id }}-{{ grains.vendor }}
</pre>

All it does it configures the hostname of the device using the Minion ID and the `vendor` Grain.

Let's execute this template and load it on the device:

```bash
salt core1 net.load_template salt://templates/iosxr.jinja test=True debug=True
```

<pre>
root@salt:~# salt core1 net.load_template salt://templates/iosxr.jinja test=True debug=True
core1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        ---
        +++
        @@ -1,6 +1,6 @@
         !! Last configuration change at Wed Jan  6 15:03:53 2021 by apnic
         !
        -hostname core1
        +hostname core1-Cisco
         interface Loopback0
         !
         interface MgmtEth0/0/CPU0/0
    loaded_config:
        hostname core1-Cisco
    result:
        True
</pre>

As one can notice in the configuration diff, loading this template, would update the hostname of `core1` to _core1-Cisco_, as this is how we configured in the Jinja template above (and pointed in the `loaded_config` key from the output, as the command has been executed with `debug=True`).

Now, for a different platform, we'd have a separate template, say for the routers level:

```bash
cat /srv/salt/templates/junos.jinja
```

<pre>
root@salt:# cat /srv/salt/templates/junos.jinja

set system host-name {{ opts.id }}-{{ grains.vendor }}
</pre>

For the spine level, `/srv/salt/templates/eos.jinja` would have, in this particular scenario, the same contents as `/srv/salt/templates/iosxr.jinja`:

```bash
salt router1 net.load_template salt://templates/junos.jinja test=True debug=True
```

<pre>
root@salt:~# salt router1 net.load_template salt://templates/junos.jinja test=True debug=True
router1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        [edit system]
        -  host-name router1;
        +  host-name router1-Juniper;
    loaded_config:
        set system host-name router1-Juniper
    result:
        True
root@salt:~# salt spine1 net.load_template salt://templates/eos.jinja test=True debug=True
spine1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -4,7 +4,7 @@
         !
         transceiver qsfp default-mode 4x10G
         !
        -hostname spine1
        +hostname spine1-Arista
         !
         spanning-tree mode mstp
         !
    loaded_config:
        hostname spine1-Arista
    result:
        True
</pre>

But wouldn't it be more flexible, and easier to use just one template for all the platforms? We can merge everything into a single template:

```bash
cat /srv/salt/templates/hostname.jinja
```

<pre>
root@salt:~# cat /srv/salt/templates/hostname.jinja
{%- if grains.os == 'junos' %}
set system host-name {{ opts.id }}-{{ grains.vendor }}
{%- else %}
hostname {{ opts.id }}-{{ grains.vendor }}
{%- endif %}
</pre>

In this short template, we can model in such a way that the same Jinja template can be used transparently against Juniper devices as well as others that share the same syntax for configuring the hostname. Therefore, we can execute the following command which would apply to all the devices simultaneously. Not only this is easier from an user perspective, but provides fluidity and consistency, while abstracting away the operations:

```bash
salt -L router1,core1,spine1,leaf1 net.load_template salt://templates/hostname.jinja test=True debug=True
```

<pre>
root@salt:~# salt -L router1,core1,spine1,leaf1 net.load_template salt://templates/hostname.jinja test=True debug=True
router1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        [edit system]
        -  host-name router1;
        +  host-name router1-Juniper;
    loaded_config:

        set system host-name router1-Juniper
    result:
        True
core1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        ---
        +++
        @@ -1,6 +1,6 @@
         !! Last configuration change at Wed Jan  6 15:03:53 2021 by apnic
         !
        -hostname core1
        +hostname core1-Cisco
         interface Loopback0
         !
         interface MgmtEth0/0/CPU0/0
    loaded_config:
        hostname core1-Cisco
    result:
        True
spine1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -4,7 +4,7 @@
         !
         transceiver qsfp default-mode 4x10G
         !
        -hostname spine1
        +hostname spine1-Arista
         !
         spanning-tree mode mstp
         !
    loaded_config:
        hostname spine1-Arista
    result:
        True
leaf1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        +hostname leaf1-Cisco
    loaded_config:
        hostname leaf1-Cisco
    result:
        True
</pre>

Notice that thanks to a simple check against the `os` Grain, we are able to model different behaviour per separate platform. The same can be extended to other similar use cases, when we need to provide different configuration based on different other factors such as OS version, chassis model, etc.

#### Scheduled commits and reverts

Beginning with Salt release 2019.2.0, both `net.load_template` and `net.load_config` functions allow scheduled commits and reverts using the `commit_at`, `commit_in`, `revert_at` and `revert_in` arguments, for any platform managed through NAPALM, regardless if the network operating system itself is capable to offer such features. The main caveat is that Salt needs to constantly have access to the network device, so if your changes are detrimental and access is lost, Salt will be able to revert the changes.

For example, schedule a commit to be executed in two minutes:

```bash
salt router1 net.load_config filename=salt://static/junos commit_in=2m
```

<pre>
root@salt:~# salt router1 net.load_config filename=salt://static/junos commit_in=2m
router1:
    ----------
    already_configured:
        False
    comment:
        Changes discarded for now, and scheduled commit at: 2021-01-06T12:29:42.
        The commit ID is: 20210106122741814566.
        To discard this commit, you can execute:

        salt router1 net.cancel_commit 20210106122741814566
    commit_id:
        20210106122741814566
    diff:
        [edit system]
        +   ntp {
        +       server 10.0.0.1;
        +   }
    loaded_config:
    result:
        True
</pre>

As the comment states, you are also able to discard the commit, by executing a specific command.

Similarly, for a revert:

```bash
salt router1 net.load_config filename=salt://static/junos debug=True revert_in=2m
```

<pre>
root@salt:~# salt router1 net.load_config filename=salt://static/junos debug=True revert_in=2m
router1:
    ----------
    already_configured:
        False
    comment:
        The commit ID is: 20210106123728539773.
        This commit will be reverted at: 2021-01-06T12:39:28, unless confirmed.
        To confirm the commit and avoid reverting, you can execute:

        salt router1 net.confirm_commit 20210106123728539773
    commit_id:
        20210106123728539773
    diff:
        [edit system]
        +   ntp {
        +       server 10.0.0.1;
        +   }
    loaded_config:
        set system ntp server 10.0.0.1
    result:
        True
</pre>

## Part-2: Configuration management using Netmiko

### Preparing the environment

Firstly, let's stop the Proxy Minions, by running the following command:

```bash
pkill -9 -e -f salt-proxy
```

Without changing anything in the Pillar Top File, we only need to update the individual Pillar file for each platform.

Update **/srv/salt/pillar/junos.sls** by completing these commands

```bash
sed -i 's/napalm/netmiko/' /srv/salt/pillar/*.sls
sed -i 's/driver/device_type/' /srv/salt/pillar/*.sls
sed -i 's/junos/juniper_junos/' /srv/salt/pillar/junos.sls
cat /srv/salt/pillar/junos.sls
```

<pre>
proxy:
  proxytype: netmiko
  device_type: juniper_junos
  host: {{ opts.id }}
  username: apnic
  password: APNIC2021
</pre>

The `proxytype` should now point to `netmiko`, while the `device_type` should be `juniper_junos` as documented in [https://docs.saltstack.com/en/master/ref/proxy/all/salt.proxy.netmiko_px.html](https://docs.saltstack.com/en/master/ref/proxy/all/salt.proxy.netmiko_px.html), as this is what Netmiko expects.

Similarly, the `device_type` will be as following:

Update the device type to cisco_xr in /srv/salt/pillar/iosxr.sls:

```bash
sed -i 's/iosxr/cisco_xr/' /srv/salt/pillar/iosxr.sls
```

Set device type to arista_eos in /srv/salt/pillar/eos.sls

```bash
sed -i 's/eos/arista_eos/' /srv/salt/pillar/eos.sls
```

Set device type to cisco_ios in /srv/salt/pillar/ios.sls

```bash
sed -i 's/ios/cisco_ios/' /srv/salt/pillar/iosxr.sls
```

With that said, we can then start the Proxy Minions for each, in the exact same way as previously:

```bash
salt-proxy --proxyid router1 -d
salt-proxy --proxyid core1 -d
salt-proxy --proxyid spine1 -d
salt-proxy --proxyid leaf1 -d
```

Then, to confirm they are all up:

```bash
salt -L router1,core1,spine1,leaf1 test.ping
```

<pre>
root@salt:~# salt -L router1,core1,spine1,leaf1 test.ping
core1:
    True
spine1:
    True
router1:
    True
leaf1:
    True
</pre>

```bash
salt -L router1,core1,spine1,leaf1 netmiko.send_command 'show version'
```

### `netmiko.send_config`

While the features available for Netmiko aren't as diverse as when working with NAPALM, we are still able to execute the **netmiko.send_config** function which accepts the config either as a template, or a list of configuration lines to be deployed on the device. This function has the following arguments:

- **config_file**: The source file with the configuration commands to be sent to the device. As with NAPALM's `net.load_config` and `net.load_template` functions, this argument accepts files specified as absolute path, or using the `salt://`, `http(s)://`, `s3://`, `ftp:/` URIs.
- **config_commands**: A list of configuration commands to deploy. The configuration lines can similarly be rendered using the template language of choice.
- **commit**: Boolean value, by default `False`, as most of the platforms managed through Netmiko don't have explicit commit capabilities. For platforms such as Cisco IOS that don't have explicit commit mechanisms (i.e., configuration is automatically deployed into the running config), this is the default implicit behaviour. **Important**: due to this Netmiko design, on WYSIWYG platforms such as Cisco IOS, there is no dry-run mechanism and everything is directly deployed into the running config.


Let's see how we can apply changes on a Junos device:

```bash
salt router1 netmiko.send_config config_commands="['set system ntp server 10.0.0.1']"
```

<pre>
root@salt:~# salt router1 netmiko.send_config config_commands="['set system ntp server 10.0.0.1']"
router1:
    configure
    Entering configuration mode
    The configuration has been changed but not committed

    [edit]
    apnic@router1# set system ntp server 10.0.0.1

    [edit]
    apnic@router1# exit configuration-mode
    The configuration has been changed but not committed
    Exiting configuration mode

    apnic@router1>
root@salt:~#
</pre>

As one can notice, unfortunately, the configuration-management capabilities of Netmiko are more limited, and the return isn't as structured as when working with NAPALM, while the config diff is not returned either. But there is an workaround that: in the list of configuration changes to be loaded, at the very end, insert the platform-specific command that will show the config diff. On Junos, that is `show | compare`:

```bash
salt router1 netmiko.send_config config_commands="['set system ntp server 10.0.0.1', 'show | compare']" commit=True
```

<pre>
root@salt:~# salt router1 netmiko.send_config config_commands="['set system ntp server 10.0.0.1', 'show | compare']" commit=True
router1:
    configure 
    Entering configuration mode
    The configuration has been changed but not committed
    
    [edit]
    apnic@router1# set system ntp server 10.0.0.1 
    
    [edit]
    apnic@router1# show | compare 
    [edit system]
    +   ntp {
    +       server 10.0.0.1;
    +   }
    
    [edit]
    apnic@router1# commit 
    commit complete
    
    [edit]
    apnic@router1# 
</pre>

We are able to re-use the previous `salt://templates/hostname.jinja` template, with one big difference: Netmiko doesn't provide the Grains in the same way as NAPALM. For this reasoning, the template must be slightly adjusted; while it's less flexible, we can use other data inputs to model the cross-platform behaviour:

```bash
cat /srv/salt/templates/hostname-netmiko.jinja
```

<pre>
root@salt:~# cat /srv/salt/templates/hostname-netmiko.jinja
{%- if pillar.proxy.device_type == 'juniper_junos' %}
set system host-name {{ opts.id }}-{{ pillar.proxy.device_type }}
{%- else %}
hostname {{ opts.id }}-{{ pillar.proxy.device_type }}
{%- endif %}
</pre>

Executing against two Cisco platforms, it returns:

```bash
salt -L core1,leaf1 netmiko.send_config salt://templates/hostname-netmiko.jinja
```

<pre>
root@salt:~# salt -L core1,leaf1 netmiko.send_config salt://templates/hostname-netmiko.jinja
core1:
    config term

    Wed Jan  6 17:57:32.139 UTC
    RP/0/0/CPU0:core1-cisco_xr(config)#hostname core1-cisco_xr

    RP/0/0/CPU0:core1-cisco_xr(config)#
leaf1:
    config term
    Enter configuration commands, one per line.  End with CNTL/Z.
    leaf1-cisco_ios(config)#hostname leaf1-cisco_ios
    leaf1-cisco_ios(config)#end
    leaf1-cisco_ios#
root@salt:~#
</pre>

## Part-3: Configuration management using PyEZ

As in the previous section, let's start by stopping the Proxy Minions:

```bash
pkill -9 -e -f salt-proxy
```

As the Junos PyEZ is focused only on Junos devices, in this scenario we are limited to using only the `router1` and `router2` devices. In that case, updating the **/srv/salt/pillar/junos.sls** file would suffice:

```bash
sed -i '/junos/d' /srv/salt/pillar/junos.sls
sed -i 's/netmiko/junos/' /srv/salt/pillar/junos.sls
cat /srv/salt/pillar/junos.sls
```

<pre>
proxy:
  proxytype: junos
  host: {{ opts.id }}
  username: apnic
  password: APNIC2021
</pre>

Afterwards, start the Proxy Minions for `router1` and `router2`, respectively:

```bash
salt-proxy --proxyid router1 -d
salt-proxy --proxyid router2 -d
```

Complete a test ping

```bash
salt router* test.ping
```

<pre>
root@salt:~# salt router* test.ping
router2:
    True
router1:
    True
</pre>

Once the Proxies are up and running, take a moment to inspect the collected Grains:

```bash
salt router* grains.items
```

In fact, the Junos Proxy Module, puts the Grains under a dedicated key, `junos_facts`, so whenever you will want to access this data, you will need to nest it under `junos_facts`, e.g.,

```bash
salt -G junos_facts:model:VMX --preview
```

<pre>
root@salt:~# salt -G junos_facts:model:VMX --preview
- router1
- router2
</pre>

#### `junos.install_config`

The `junos.install_config` function requires the configuration to be provided through a file. If the file has a `.conf` extension, the content is treated as text format. If the file has a `.xml` extension, the content is treated as XML format. If the file has a `.set` extension, the content is treated as Junos OS `set` commands.

Other optional arguments include:

- **mode**: The mode in which the configuration is locked. Can be one of `private`, `dynamic`, `batch`, `exclusive`. Default: `exclusive`.
- **replace**: Specify whether the configuration file uses `replace:` statements. If `True`, only those statements under the `replace` tag will be changed.
- **format**: Determines the format of the contents.
- **comment**: Provide a comment for the commit.
- **confirm**: Provide time in minutes for commit confirmation. If this option is specified, the commit will be rolled back in the specified amount of time unless the commit is confirmed.
- **template_vars**: Variables to be passed into the template processing engine in addition to those present in pillar, the minion configuration, grains, etc.

For example, let's reuse the static file from _Part-1_, `/srv/salt/static/junos`. As a reminder these are the contents:

```bash
cat /srv/salt/static/junos
```

<pre>
root@salt:~# cat /srv/salt/static/junos
set system ntp server 10.0.0.1
root@salt:~#
</pre>

As the file is located under the Salt file system, we can reference it as `salt://static/junos`:

```bash
salt router* junos.install_config salt://static/junos format=set
```

<pre>
root@salt:~# salt router* junos.install_config salt://static/junos format=set
router1:
    ----------
    message:
        Successfully loaded and committed!
    out:
        True
router2:
    ----------
    message:
        Successfully loaded and committed!
    out:
        True
</pre>

Loading the configuration, and specifying that the contents as `set`, will configure the device and commit the results. The Junos module doesn't have capabilities to execute a dry-run before deploying the configuration to check the diff. It does however provide the individual features in order to somewhat reproduce what you could do from your network device command line. This means, a multi-step procedure:

1. Load the configuration, while scheduled a revert (i.e., _commit confirmed_), for example in 1 minute. At the same time, save the diffs into a file.

   ```bash
   salt router1 junos.install_config salt://static/junos format=set diffs_file=/tmp/diffs confirm=1
   ```
   
   <pre>
   root@salt:~# salt router1 junos.install_config salt://static/junos format=set diffs_file=/tmp/diffs confirm=1
    router1:
        ----------
        message:
            Successfully loaded and committed!
        out:
            True
    </pre>

3. Inspect the contents of the diffs.

   ```bash
   cat /tmp/diffs
   ```

   <pre>
   root@salt:~# cat /tmp/diffs
   
   [edit system]
   +   ntp {
   +       server 10.0.0.1;
   +   }
   </pre>

4. If the config diff looks good, commit so the device won't revert the changes.

   ```bash
   salt router1 junos.commit
   ```

   <pre>
   root@salt:~# salt router1 junos.commit
   router1:
       ----------
       message:
           Commit Successful.
       out:
           True
    </pre>

In addition to these function, the Junos module provides for configuration management:

- **junos.commit_check**: Perform a commit check on the configuration.
- **junos.diff**: Returns the difference between the candidate and the current configuration.
- **junos.rollback**: Roll back the last committed configuration changes and commit.
- **junos.set_hostname**: Set the device's hostname.

With these, we can consider that we now have the fundamentals to be managing configuration using Salt, for widely-used native modules.

---
**End of Lab**

---
