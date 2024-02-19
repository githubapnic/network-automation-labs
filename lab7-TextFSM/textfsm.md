![](images/apnic_logo.png)
# LAB: Parsing Text Output using TextFSM

For simplicity, all the Proxy Minions for this lab are already started, using the NAPALM Proxy Module. The topic covered 
in the _Part-1_ however are widely available to any Minion flavour, as long as the `textfsm` library is installed.

## Part-1: Using TextFSM through Salt

In this section, we will be using the 
[`textfsm`](https://docs.saltstack.com/en/master/ref/modules/all/salt.modules.textfsm_mod.html) Salt module which can be 
used to apply TextFSM operations using arbitrary input text and FSM templates. There are two functions available in this 
module: `textfsm.extract` and `textfsm.index`:

- `textfsm.extract` applies the TextFSM template on a text (either provided via the CLI or through a file), then returns 
  the data it extracted.
- `testfsm.index` dynamically determines what TextFSM template to apply to a specific text output, based on the various 
  parameters (command, platform, etc.). The value of this functionality is better understood through cross-vendor 
  examples, as we will see in _Part-2_ of this lab, so we won't focus on this just yet.

### `textfsm.extract`

The `textfsm.extract` function accepts the following arguments:

- `template_path`: The path to the TextFSM template. This can either be an absolute path or using on the usual URI 
  schemes `salt://`, `http(s)://`, `s3://`, `ftp://`, `swift://`, etc.
- `raw_text`: The input text where to extract the data from.
- `raw_text_file`: The file having the text contents to read from. Supports the same URI schemes as `template_path`.
- `saltenv`: The name of the Salt environment (particularly useful when using the `salt://` URI scheme).

Let's have a file with the following contents:

`/srv/salt/textfsm/eos_show_version.txt`

```
Arista vEOS
Hardware version:
Serial number:
System MAC address:  5254.0087.87be

Software image version: 4.18.1F
Architecture:           i386
Internal build version: 4.18.1F-4591672.4181F
Internal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed

Uptime:                 3 days, 23 hours and 18 minutes
Total memory:           1893316 kB
Free memory:            609540 kB
```

The TextFSM template, as discussed in the _Module 9_ slides is:

`/srv/salt/textfsm/eos_show_version.fsm`

```
Value MODEL (\S*)
Value HARDWARE_VERSION (\S+)
Value SERIAL (\S*)
Value SYSTEM_MAC (\S*)
Value SOFTWARE_VERSION (\S*)

Start
  ^Arista ${MODEL}
  ^Hardware\s*version:\s+${HARDWARE_VERSION}
  ^Serial\s*number:\s+${SERIAL}
  ^System\s*MAC\s*address:\s+${SYSTEM_MAC}
  ^Software\s*image version:\s+${SOFTWARE_VERSION} -> Record
```

As both files are located under the Salt filesystem, we can use the `salt://` URI scheme, so we can execute:

```bash
root@salt:~# salt router1 textfsm.extract salt://textfsm/eos_show_version.fsm raw_text_file=salt://textfsm/eos_show_version.txt
router1:
    ----------
    comment:
    out:
        |_
          ----------
          hardware_version:
          model:
              vEOS
          serial:
          software_version:
              4.18.1F
          system_mac:
              5254.0087.87be
    result:
        True
```

The return is nothing else than a structured object, a list of Python dictionaries, more specifically:

```bash
root@salt:~# salt router1 textfsm.extract salt://textfsm/eos_show_version.fsm raw_text_file=salt://textfsm/eos_show_version.txt --out=raw
{'router1': {'result': True, 'comment': '', 'out': [{'model': 'vEOS', 'hardware_version': '', 'serial': '', 'system_mac': '5254.0087.87be', 'software_version': '4.18.1F'}]}}
```

The actual TextFSM parsing result is nested under the `out` key, while the others, `result` and `comment` respectively 
are helpers that tell whether the parsing succeeded.

For example, if we were to provide an incorrect path, we'd get the following return:

```bash
root@salt:~# salt router1 textfsm.extract salt://textfsm/fake raw_text_file=salt://textfsm/fake
router1:
    ----------
    comment:
        Unable to read the TextFSM template from salt://textfsm/fake
    out:
        None
    result:
        False
ERROR: Minions returned with non-zero exit code
```

Similarly, when the TextFSM template exists, but there's a coding error. Let's remove the first line from 
`/srv/salt/textfsm/eos_show_version.fsm` (the definition of the `MODEL` variable):

```
Value HARDWARE_VERSION (\S+)
Value SERIAL (\S*)
Value SYSTEM_MAC (\S*)
Value SOFTWARE_VERSION (\S*)

Start
  ^Arista ${MODEL}
  ^Hardware\s*version:\s+${HARDWARE_VERSION}
  ^Serial\s*number:\s+${SERIAL}
  ^System\s*MAC\s*address:\s+${SYSTEM_MAC}
  ^Software\s*image version:\s+${SOFTWARE_VERSION} -> Record
```

Re-running the previous command, it would return:

```bash
root@salt:~# salt router1 textfsm.extract salt://textfsm/eos_show_version.fsm raw_text_file=salt://textfsm/eos_show_version.txt
router1:
    ----------
    comment:
        Unable to parse the TextFSM template from salt://textfsm/eos_show_version.fsm: Duplicate or invalid variable substitution: '^Arista ${MODEL}'. Line: 7.. Please check the logs.
    out:
        None
    result:
        False
ERROR: Minions returned with non-zero exit code
```

As the `textfsm` module is available from any Minion type (and, implicitly, any Proxy Minion - thus the Netmiko and 
Junos we've visited already), we can use it in conjunction with any function that retrieves CLI output from the network 
devices - e.g., `netmiko.send_command`, or `junos.cli`, etc., then send that output to the `textfsm` module to extract 
the information required.

## Part-2: Extracting information from show-commands using the `net.cli` function

For NAPALM Proxy Minions, the function `net.cli` can be used to execute show commands and return the output as text, for 
example:

```bash
root@salt:~# salt spine1 net.cli "show version"
spine1:
    ----------
    comment:
    out:
        ----------
        show version:
            Arista vEOS
            Hardware version:
            Serial number:
            System MAC address:  5254.00e3.d6d3

            Software image version: 4.18.1F
            Architecture:           i386
            Internal build version: 4.18.1F-4591672.4181F
            Internal build ID:      6fcb426e-70a9-48b8-8958-54bb72ee28ed

            Uptime:                 5 hours and 0 minutes
            Total memory:           1893316 kB
            Free memory:            797532 kB

    result:
        True
```

`net.cli` can execute one or more CLI commands at the same time:

```bash
root@salt:~# salt spine1 net.cli "show clock" "show users"
spine1:
    ----------
    comment:
    out:
        ----------
        show clock:
            Mon Jan 11 18:17:52 2021
            Timezone: UTC
            Clock source: local
        show users:
                Line      User        Host(s)       Idle        Location 
               1 con 0    admin       idle          05:05:00    -        
               2 vty 6    apnic       idle          00:00:53    10.0.0.2 
            
    result:
        True
```

Notice the CLI results nested under each command.

It is also smart enough to execute the TextFSM templates dynamically and extract the data from the CLI output:

```bash
root@salt:~# salt spine1 net.cli "show version" textfsm_parse=True textfsm_template=salt://textfsm/eos_show_version.fsm
spine1:
    ----------
    comment:
    out:
        ----------
        show version:
            |_
              ----------
              hardware_version:
              model:
                  vEOS
              serial:
              software_version:
                  4.18.1F
              system_mac:
                  5254.00e3.d6d3
    result:
        True
```

The command executed above is simply the previous command, just with the `textfsm_parse` and `textfsm_template` 
arguments passed in: `textfsm_parse` tells whether we want data extraction through TextFSM and the `textfsm_template` 
provides the location of the template, as in _Part-1_.

Once again, you can verify that the output is nothing else than just a Python object:

```bash
root@salt:~# salt spine1 net.cli "show version" textfsm_parse=True textfsm_template=salt://textfsm/eos_show_version.fsm --out=raw
{'spine1': {'comment': '', 'result': True, 'out': {'show version': [{'model': 'vEOS', 'hardware_version': '', 'serial': '', 'system_mac': '5254.00e3.d6d3', 'software_version': '4.18.1F'}]}}}
```

But what if we want to extract data from multiple show-commands in one go:

```
root@salt:~# salt spine1 net.cli "show version" "show clock" textfsm_parse=True textfsm_template=salt://textfsm/eos_show_version.fsm
spine1:
    ----------
    comment:
    out:
        ----------
        show clock:
            Mon Jan 11 18:26:55 2021
            Timezone: UTC
            Clock source: local
        show version:
            |_
              ----------
              hardware_version:
              model:
                  vEOS
              serial:
              software_version:
                  4.18.1F
              system_mac:
                  5254.00e3.d6d3
    result:
        True
```

Executing the same template against multiple sources doesn't error, but it doesn't provide the desired result either.

Let's define a TextFSM template corresponding to the output of the _show clock_ command:

`/srv/salt/textfsm/eos_show_clock.fsm`

```
Value DAY_NAME (\w+)
Value MONTH (\w+)
Value DAY (\d+)
Value TIME (\d+:\d+:\d+)
Value YEAR (\d+)
Value TIMEZONE (\S+)
Value SOURCE (\S+)

Start
  ^${DAY_NAME}\s+${MONTH}\s+${DAY}\s+${TIME}\s+${YEAR}
  ^Timezone:\s+${TIMEZONE}
  ^Clock source:\s+${SOURCE}
```

Take a moment to understand each configuration bit from this template.

To verify that it works as expected, run:

```bash
root@salt:~# salt spine1 net.cli "show clock" textfsm_parse=True textfsm_template=salt://textfsm/eos_show_clock.fsm
spine1:
    ----------
    comment:
    out:
        ----------
        show clock:
            |_
              ----------
              day:
                  11
              day_name:
                  Mon
              month:
                  Jan
              source:
                  local
              time:
                  18:34:54
              timezone:
                  UTC
              year:
                  2021
    result:
        True
```

Running this template against the _show version_ and _show clock_ commands, we bump into the same issue, as previously. 
But there's a way to work around this, by using the `textfsm_template_dict` which is a nice way to define which TextFSM 
template gets applied to which show command:

```bash
root@salt:~# salt spine1 net.cli "show clock" "show version" textfsm_parse=True \
textfsm_template_dict="{'show clock': 'salt://textfsm/eos_show_clock.fsm', 'show version': 'salt://textfsm/eos_show_version.fsm'}"
spine1:
    ----------
    comment:
    out:
        ----------
        show clock:
            |_
              ----------
              day:
                  11
              day_name:
                  Mon
              month:
                  Jan
              source:
                  local
              time:
                  18:42:38
              timezone:
                  UTC
              year:
                  2021
        show version:
            |_
              ----------
              hardware_version:
              model:
                  vEOS
              serial:
              software_version:
                  4.18.1F
              system_mac:
                  5254.00e3.d6d3
    result:
        True
```

But this is rather boring to type (and error prone too!), so there's an easier way: simply define show command--TextFSM 
template mapping into the Pillar. As this is platform-specific, we can place this under the Pillar where we have defined 
the Proxy authentication details - in this case, the `/srv/salt/pillar/eos.sls` Pillar file:

```yaml
napalm_cli_textfsm_template_dict:
  "show clock": "salt://textfsm/eos_show_clock.fsm"
  "show version": "salt://textfsm/eos_show_version.fsm"
```

Refresh the Pillar data to ensure the new configuration is loaded:

```bash
root@salt:~# salt spine1 saltutil.refresh_pillar
spine1:
    True
root@salt:~# salt spine1 config.get napalm_cli_textfsm_template_dict
spine1:
    ----------
    show clock:
        salt://textfsm/eos_show_clock.fsm
    show version:
        salt://textfsm/eos_show_version.fsm
```

With that configured, we can have `net.cli` without having to provide all the details from the command line:

```bash
root@salt:~# salt spine1 net.cli "show clock" "show version" textfsm_parse=True
spine1:
    ----------
    comment:
    out:
        ----------
        show clock:
            |_
              ----------
              day:
                  11
              day_name:
                  Mon
              month:
                  Jan
              source:
                  local
              time:
                  18:49:45
              timezone:
                  UTC
              year:
                  2021
        show version:
            |_
              ----------
              hardware_version:
              model:
                  vEOS
              serial:
              software_version:
                  4.18.1F
              system_mac:
                  5254.00e3.d6d3
    result:
        True
```

Even more, by setting the `napalm_cli_textfsm_parse` flag into the same Pillar file:

`/srv/salt/pillar/eos.sls`

```yaml
napalm_cli_textfsm_parse: true
```

After Pillar refresh, we can execute directly as `salt spine1 net.cli "show clock" "show version"` and the output will 
always be parsed and the data extracted for the two show commands.

### Using the `index` file

Using the `textfsm_template_dict` argument is a good way to map the TextFSM to use for each show command, but on the 
flip side, it requires to always execute the exact command as defined in the mapping - for example, if you type _sh ver_ 
instead of _show version_, it won't work so well.

Here's why TextFSM has an `index` file where we can reference which templates to be invoked to which show commands:

`/srv/salt/textfsm/index`

```
Template, Hostname, Platform, Command

eos_show_version.fsm, .*, eos, sh[[ow]] ver[[sion]]
```

The first line of the file is the header, where we define the column names - Template, Hostname, Platform and Command. 
This is a very granular way to select which template is applied to what command output, platform and hostname.

The next line(s) represent(s) the mapping itself; the line reads as: "on any EOS device, any hostname, apply the 
`eos_show_version.fsm` on any variation of the *show version* command - i.e., _sh ver_, _sho vers_, etc.").

With the _index_ file defined in the same place as the TextFSM templates, we can execute:

```bash
root@salt:~# salt spine1 net.cli "show version" textfsm_parse=True textfsm_path=salt://textfsm/
spine1:
    ----------
    comment:
    out:
        ----------
        show version:
            |_
              ----------
              hardware_version:
              model:
                  vEOS
              serial:
              software_version:
                  4.18.1F
              system_mac:
                  5254.00e3.d6d3
    result:
        True
```

We can similarly set this into the Pillar and simplify the usage:

`/srv/salt/pillar/eos.sls`

```yaml
textfsm_path: salt://textfsm/
```

With that value set, the execution becomes as simple as `salt spine1 net.cli "show version" textfsm_parse=True` with the 
`textfsm_parse` flag being toggled whenever we want the output parsed or not.

The _index_ file is equally a boon for resolving cross-vendor representation. For example, to have the _show version_ 
command parsed on any platform, the file becomes:

`/srv/salt/textfsm/index`

```
Template, Hostname, Platform, Command

eos_show_version.fsm, .*, eos, sh[[ow]] ver[[sion]]
junos_show_version.fsm, .*, junos, sh[[ow]] ver[[sion]]
ios_show_version.fsm, .*, ios, sh[[ow]] ver[[sion]]
iosxr_show_version.fsm, .*, iosxr, sh[[ow]] ver[[sion]]
```

You can find the FSM files on the server, under the `/srv/salt/textfsm` path. With that, we can run the same command to 
return the structured data from any platform:

```bash
root@salt:~# salt \* net.cli "show version" textfsm_parse=True textfsm_path=salt://textfsm/
Executing job with jid 20210111192750452014
-------------------------------------------

core1:
    ----------
    comment:
    out:
        ----------
        show version:
            |_
              ----------
              config_register:
              hardware:
              hostname:
                  core1
              mac:
              reload_reason:
              rommon:
                  GRUB,
              running_image:
                  disk0/xrvr-os-mbi-6.0.0/mbixrvr-rp.vm
              serial:
              uptime:
                  6 hours, 17 minutes
              version:
    result:
        True
leaf1:
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
                  leaf1
              mac:
              reload_reason:
                  <NULL>
              rommon:
                  IOS-XE
              running_image:
                  packages.conf
              serial:
                  - 9E5T90JKB15
              uptime:
                  6 hours, 15 minutes
              version:
                  15.5(2)S
    result:
        True

...
... snip ...
...
```

---
**End of Lab**

---
