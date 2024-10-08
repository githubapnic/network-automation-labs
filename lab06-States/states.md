![](images/apnic_logo.png)
# LAB: Salt States: Advanced Configuration Management

In this lab we will continue the exercise from _Lab 5_ and continue from there with the configuration management capabilities of Junos and NAPALM through the State system. As of present, Netmiko doesn't provide a State module to manage the configuration, however the function we've seen earlier, `netmiko.send_config` can be accessed through the `module.run` State function (see https://docs.saltstack.com/en/master/ref/states/all/salt.states.module.html for further details on this particular topic).

## Part-1: The Junos State Module

The documentation for this State Module is available at [https://docs.saltstack.com/en/master/ref/states/all/salt.states.junos.html](https://docs.saltstack.com/en/master/ref/states/all/salt.states.junos.html).

As the previous lab concluded with using the Junos module, the `/srv/salt/pillar/junos.sls` Pillar file should still have the following content:

```bash
cat /srv/salt/pillar/junos.sls
```

<pre>
proxy:
  proxytype: junos
  host: {{ opts.id }}
  username: admin
  password: admin@123
</pre>

Similarly, the Proxy Minions should also be started up:

```bash
ps -aux | grep -v defunct | grep proxy
```

If the router1 and router2 Proxy Minions are not running start them with the following commands

<pre>
root@salt:~# salt-proxy --proxyid router1 -d
root@salt:~# salt-proxy --proxyid router2 -d
root@salt:~#
root@salt:~# salt router* test.ping
router2:
    True
router1:
    True
</pre>

Now, under the `/srv/salt/states` directory we can place our first State SLS file, e.g., `hostname.sls`, where we can put the State declarations, for example:

```bash
cat <<EOF > /srv/salt/states/hostname.sls
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
EOF
```

<pre>
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
</pre>

The State name is _Configure hostname_, and it invokes the `junos.install_config` State function.

**Important**: The State function `junos.install_config` coincidentally has the same naming as the Execution Function we've used in _Lab 5_, in this particular case. This isn't always the case, and you should always refer to the documentation to make sure.

The arguments `name` is the configuration file path and `format` is the same as we've used previously. As a reminder, we've previously executed:

<pre>
root@salt:~# salt router* junos.install_config salt://static/junos format=set
router1:
    ----------
    message:
        Successfully loaded and committed!
    out:
        True
</pre>

Through the State system now, we can execute with the same effect:

```bash
salt router1 state.apply hostname
```

<pre>
root@salt:~# salt router1 state.apply hostname
router1:
----------
          ID: Configure hostname
    Function: junos.install_config
        Name: salt://static/junos
      Result: True
     Comment:
     Started: 18:16:52.293257
    Duration: 1522.678 ms
     Changes:
              ----------
              message:
                  Successfully loaded and committed!
              out:
                  True

Summary for router1
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:   1.523 s
</pre>

This has a number of advantages over the CLI usage:

- You don't need to remember the syntax of the CLI function, but only configure it one in the SLS file. Instead, you only need to know the name of the State to invoke to deploy the desired configuration bit.
- The syntax of the CLI function can change from one Salt release to another.
- The output is more human friendly and provides additional details.

The most important aspect however is that through the State system, you can chain multiple States and build complex workflows. For example, let's update the state to save the configuration diff into a file. As with the `junos.install_config` function, you can use the `diffs_file` argument:

Update the /srv/salt/states/hostname.sls file with:

```bash
cat <<EOF > /srv/salt/states/hostname.sls
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
    - diffs_file: /tmp/diff

Display diffs:
  cmd.run:
    - name: cat /tmp/diff
    - onsuccess:
      - junos: Configure hostname
EOF
```

<pre>
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
    - diffs_file: /tmp/diff

Display diffs:
  cmd.run:
    - name: cat /tmp/diff
    - onsuccess:
      - junos: Configure hostname
</pre>

Besides the _Configure hostname_, there's another State defined in the SLS file: _Display diffs_, which executes the `cmd.run` State function: see [https://docs.saltstack.com/en/latest/ref/states/all/salt.states.cmd.html](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.cmd.html) for more details. This State runs a simple shell command, `cat /tmp/diff`, in order to display the contents of the `/tmp/diff` file, where the config diffs have been saved. This State however is executed _only_ when _Configure hostname_ is applied successfully.

Firstly let's rollback the changes to the previous state:

```bash
salt router1 junos.rollback
```

<pre>
root@salt:~# salt router1 junos.rollback
router1:
    ----------
    message:
        Rollback successful
    out:
        True
</pre>

Now, run the State:

```bash
salt router1 state.apply hostname
```

<pre>
root@salt:~# salt router1 state.apply hostname
router1:
----------
          ID: Configure hostname
    Function: junos.install_config
        Name: salt://static/junos
      Result: True
     Comment:
     Started: 18:30:47.825363
    Duration: 1350.41 ms
     Changes:
              ----------
              message:
                  Successfully loaded and committed!
              out:
                  True
----------
          ID: Display diffs
    Function: cmd.run
        Name: cat /tmp/diff
      Result: True
     Comment: Command "cat /tmp/diff" run
     Started: 18:30:49.176353
    Duration: 10.301 ms
     Changes:
              ----------
              pid:
                  5602
              retcode:
                  0
              stderr:
              stdout:

                  [edit system]
                  +   ntp {
                  +       server 10.0.0.1;
                  +   }

Summary for router1
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   1.361 s
</pre>

This way, just by running a command, we are able to execute a sequence of tasks. Let's take this further and execute the previous 3-step commit-confirm, diff check and commit from _Lab 5_. For this, say we put the expected diff into a file, say `/srv/salt/static/junos.diff` with this content:

```bash
cat /srv/salt/static/junos.diff
```

<pre>
[edit system]
+   ntp {
+       server 10.0.0.1;
+   }
</pre>

For this, we will need a way to compare the two files. For this task, we can use the [`file.managed`](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html#salt.states.file.managed) State function, which is one of the most widely used for managing the contents of the `/srv/salt/states/hostname.sls`

<pre>
Check diffs:
  file.managed:
    - name: /tmp/diff
    - source: salt://static/junos.diff
    - onsuccess:
      - junos: Configure hostname
</pre>

This State would only be executed on successful configuration changes, and it would attempt to bring the file `/tmp/diff` to have the contents from `/srv/salt/static/junos.diff`. If there are changes, then the diff isn't as we expected it to be, so we shouldn't commit, but rather error and stop the execution. Using the `onchanges` requisite, we can do this exactly like this:

<pre>
Configuration differs:
  test.fail_without_changes:
    - onchanges:
      - file: Check diffs
</pre>

The _Configuration differs_, which marks a failure, would be executed only when _Check diffs_ returns changes.

And, finally, the _Commit config_ State:

<pre>
Commit config:
  junos.commit:
    - require:
      - test: Configuration differs
</pre>

This one, through the `require` keyword, would only be executed when _Configuration differs_ "succeeds" (or, better worded it is not executed, as if it is executed, it is a failure).

To sum it up, here is the complete `hostname.sls` State SLS:


<pre>
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
    - diffs_file: /tmp/diff
    - confirm: 1

Display diffs:
  cmd.run:
    - name: cat /tmp/diff
    - onsuccess:
      - junos: Configure hostname

Check diffs:
  file.managed:
    - name: /tmp/diff
    - source: salt://static/junos.diff
    - onsuccess:
      - junos: Configure hostname

Configuration differs:
  test.fail_without_changes:
    - onchanges:
      - file: /tmp/diff

Commit config:
  junos.commit:
    - require:
      - test: Configuration differs
</pre>

Create the hostname.sls with these settings

```bash
cat <<EOF > /srv/salt/states/hostname.sls
Configure hostname:
  junos.install_config:
    - name: salt://static/junos
    - format: set
    - diffs_file: /tmp/diff
    - confirm: 1

Display diffs:
  cmd.run:
    - name: cat /tmp/diff
    - onsuccess:
      - junos: Configure hostname

Check diffs:
  file.managed:
    - name: /tmp/diff
    - source: salt://static/junos.diff
    - onsuccess:
      - junos: Configure hostname

Configuration differs:
  test.fail_without_changes:
    - onchanges:
      - file: /tmp/diff

Commit config:
  junos.commit:
    - require:
      - test: Configuration differs
EOF
```

Run the state apply command again

```bash
salt router1 state.apply hostname
```

<pre>
root@salt:~# salt router1 state.apply hostname
router1:
----------
          ID: Configure hostname
    Function: junos.install_config
        Name: salt://static/junos
      Result: True
     Comment:
     Started: 08:59:07.527369
    Duration: 482.411 ms
     Changes:
              ----------
              message:
                  Configuration already applied!
              out:
                  True
----------
          ID: Display diffs
    Function: cmd.run
        Name: cat /tmp/diff
      Result: True
     Comment: Command "cat /tmp/diff" run
     Started: 08:59:08.010675
    Duration: 11.349 ms
     Changes:
              ----------
              pid:
                  134269
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Check diffs
    Function: file.managed
        Name: /tmp/diff
      Result: True
     Comment: File /tmp/diff updated
     Started: 08:59:08.027769
    Duration: 36.298 ms
     Changes:
              ----------
              diff:
                  ---
                  +++
                  @@ -0,0 +1,5 @@
                  +
                  +[edit system]
                  ++   ntp {
                  ++       server 10.0.0.1;
                  ++   }
----------
          ID: Configuration differs
    Function: test.fail_without_changes
      Result: False
     Comment: Failure!
     Started: 08:59:08.065952
    Duration: 1.421 ms
     Changes:
----------
          ID: Commit config
    Function: junos.commit
      Result: False
     Comment: One or more requisite failed: hostname.Configuration differs
     Started: 08:59:08.067712
    Duration: 0.003 ms
     Changes:

Summary for router1
------------
Succeeded: 3 (changed=3)
Failed:    2
------------
Total states run:     5
Total run time: 531.482 ms
</pre>

This is one of the greatest powers of the State system: building complex workflows. There are other important aspects such as parallelization and execution queueing, but that is beyond the current scope.


## Part-2: The NetConfig State module

The _NetConfig_ State module is based on NAPALM's capabilities explored in _Lab 5_, and therefore inherits all the features and simplicity. For instance, instead of having a series of states just for checking the diff, we can just execute the state in dry-run mode, by simply passing in the `test=True` flag on the command line.

For simplicity, let's stop the running Proxy Minions:

```bash
pkill salt-proxy
```

In the meantime, the Proxy Minions for all the devices will be started up as Docker containers, and you should be ready to use them:

```bash
salt \* test.ping
```

<pre>
root@salt:~# salt \* test.ping
leaf3:
    True
spine1:
    True
spine2:
    True
core1:
    True
spine4:
    True
core2:
    True
spine3:
    True
leaf4:
    True
leaf1:
    True
leaf2:
    True
router1:
    True
router2:
    True
</pre>

Update the /srv/salt/states/hostname.sls to use the jinja template:


```bash
cat <<EOF > /srv/salt/states/hostname.sls
Configure hostname:
  netconfig.managed:
    - template_name: salt://templates/hostname.jinja
EOF
```

<pre>
Configure hostname:
  netconfig.managed:
    - template_name: salt://templates/hostname.jinja
</pre>

The State is referencing the `salt://templates/hostname.jinja`, which has been defined earlier in _Lab 5_:

```bash
cat /srv/salt/templates/hostname.jinja
```

<pre>
{%- if grains.os == 'junos' %}
set system host-name {{ opts.id }}-{{ grains.vendor }}
{%- else %}
hostname {{ opts.id }}-{{ grains.vendor }}
{%- endif %}
</pre>

Ensure the proxytype is set to napalm and the driver is correct for the various vendors

```bash
grep -in napalm -A 1 /srv/salt/pillar/*.sls
```

<pre>
root@salt:~# grep -in napalm -A 1 /srv/salt/pillar/*.sls
/srv/salt/pillar/eos.sls:2:  proxytype: napalm
/srv/salt/pillar/eos.sls-3-  driver: eos
--
/srv/salt/pillar/ios.sls:2:  proxytype: napalm
/srv/salt/pillar/ios.sls-3-  driver: ios
--
/srv/salt/pillar/iosxr.sls:2:  proxytype: napalm
/srv/salt/pillar/iosxr.sls-3-  driver: iosxr
--
/srv/salt/pillar/junos.sls:2:  proxytype: napalm
/srv/salt/pillar/junos.sls-3-  driver: junos
</pre>

**Note**: Please inform the instructor if you do not see this output.

This is all required for now, and we can then execute a dry-run to check the diffs:

```bash
salt \* state.apply hostname test=True
```

<pre>
root@salt:~# salt \* state.apply hostname test=True

router1:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: None
     Comment: Configuration discarded.
              
              Configuration diff:
              
              [edit system]
              -  host-name router1;
              +  host-name router1-Juniper;
     Started: 13:46:54.064685
    Duration: 494.912 ms
     Changes:   

Summary for router1
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time: 494.912 ms
core2:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: None
     Comment: Configuration discarded.
              
              Configuration diff:
              
              --- 
              +++ 
              @@ -1,6 +1,6 @@
               !! Last configuration change at Fri Jan  8 13:38:44 2021 by apnic
               !
              -hostname core2
              +hostname core2-Cisco
               interface MgmtEth0/0/CPU0/0
                ipv4 address 10.0.0.15 255.255.255.0
               !
     Started: 13:46:53.139321
    Duration: 3336.675 ms
     Changes:   

Summary for core2
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time:   3.337 s
spine3:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: None
     Comment: Configuration discarded.
              
              Configuration diff:
              
              @@ -4,7 +4,7 @@
               !
               transceiver qsfp default-mode 4x10G
               !
              -hostname spine3
              +hostname spine3-Arista
               !
               spanning-tree mode mstp
               !
     Started: 13:46:53.182913
    Duration: 4114.652 ms
     Changes:   

Summary for spine3
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time:   4.115 s
</pre>

Notice the color scheme on the command line: the output is mostly yellow, due to the `test=True` usage to mark that this is a dry-run and then changes have been reverted, the `unchanged=1` flag in the `Succeeded` section, as well as the _Configuration discarded._ message in the `Comment`. Now, if we drop the `test=True` flag and execute again, it will effectively apply the changes on the devices and return:

```bash
salt \* state.apply hostname
```

<pre>
root@salt:~# salt \* state.apply hostname
router2:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Configuration changed!
     Started: 13:49:39.868177
    Duration: 1088.092 ms
     Changes:
              ----------
              diff:
                  [edit system]
                  -  host-name router2;
                  +  host-name router2-Juniper;

Summary for router2
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:   1.088 s
core1:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Configuration changed!
     Started: 13:49:38.928127
    Duration: 2445.287 ms
     Changes:   
              ----------
              diff:
                  --- 
                  +++ 
                  @@ -1,6 +1,6 @@
                   !! Last configuration change at Fri Jan  8 13:38:43 2021 by apnic
                   !
                  -hostname core1
                  +hostname core1-Cisco
                   interface Loopback0
                   !
                   interface MgmtEth0/0/CPU0/0

Summary for core1
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:   2.445 s
spine3:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Configuration changed!
     Started: 13:49:39.861191
    Duration: 4483.18 ms
     Changes:   
              ----------
              diff:
                  @@ -4,7 +4,7 @@
                   !
                   transceiver qsfp default-mode 4x10G
                   !
                  -hostname spine3
                  +hostname spine3-Arista
                   !
                   spanning-tree mode mstp
                   !

Summary for spine3
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:   4.483 s
leaf2:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Configuration changed!
     Started: 13:49:38.934692
    Duration: 17718.097 ms
     Changes:   
              ----------
              diff:
                  +hostname leaf2-Cisco

Summary for leaf2
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:  17.718 s
</pre>

One interesting detail to notice is the _Total run time_ which depends on the platform being executed on and the transport mechanism being used in order to speak to the network device (i.e., it's slower on Cisco and in general devices that don't have a proper API, where the communication channel is established via SSH / screen scraping, and faster on the others).

Let's run again the same command:

```bash
salt \* state.apply hostname
```

<pre>
root@salt:~# salt \* state.apply hostname
Executing job with jid 20210108135454878431
-------------------------------------------

router2:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Already configured.
     Started: 13:54:56.065097
    Duration: 524.542 ms
     Changes:

Summary for router2
------------
Succeeded: 1
Failed:    0
------------
Total states run:     1
Total run time: 524.542 ms
core1:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Already configured.
     Started: 13:54:56.067355
    Duration: 2654.425 ms
     Changes:   

Summary for core1
------------
Succeeded: 1
Failed:    0
------------
Total states run:     1
Total run time:   2.654 s
spine4:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Already configured.
     Started: 13:54:55.128703
    Duration: 4212.222 ms
     Changes:   

Summary for spine4
------------
Succeeded: 1
Failed:    0
------------
Total states run:     1
Total run time:   4.212 s
</pre>

The output is slightly different now, the state still succeeds, but without the `changed=1` flag previously seen. Also, the comment says _Already configured_.

Let's make it more interesting and have the `hostname.sls` State save a backup whenever there are changes. For this, we will need our template to do something different, so let's strip the vendor part from the configured hostname, so the template should look just like this:

```bash
sed -i 's/\-{{ grains.vendor }}//g' /srv/salt/templates/hostname.jinja
cat /srv/salt/templates/hostname.jinja
```

<pre>
{%- if grains.os == 'junos' %}
set system host-name {{ opts.id }}
{%- else %}
hostname {{ opts.id }}
{%- endif %}
</pre>

Now, let's update the `hostname.sls`, and add another state to backup the configuration:

```bash
cat <<EOF >> /srv/salt/states/hostname.sls

Backup config:
  netconfig.saved:
    - name: /srv/salt/bkups/{{ grains.id }}.conf
    - source: running
    - makedirs: True
    - onchanges:
      - netconfig: Configure hostname
EOF
```


<pre>
Configure hostname:
  netconfig.managed:
    - template_name: salt://templates/hostname.jinja

Backup config:
  netconfig.saved:
    - name: /srv/salt/bkups/{{ grains.id }}.conf
    - source: running
    - makedirs: True
    - onchanges:
      - netconfig: Configure hostname
</pre>

The latter State, named _Backup config_ saves the running configuration of each device, into a file under the `/srv/salt/bkups/` directory, whose naming depends on the Minion ID (`grains.id`) - but _only_ when the _Configure hostname_ State renders changes, thanks to the `onchanges` keyword.

Running the state:

```bash
salt \* state.apply hostname
```

<pre>
root@salt:~# salt \* state.apply hostname
router2:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Configuration changed!
     Started: 14:13:24.976070
    Duration: 1942.503 ms
     Changes:
              ----------
              diff:
                  [edit system]
                  -  host-name router2-Juniper;
                  +  host-name router2;
----------
          ID: Backup config
    Function: netconfig.saved
        Name: /srv/salt/bkups/router2.conf
      Result: True
     Comment: File /srv/salt/bkups/router2.conf updated
     Started: 14:13:26.918830
    Duration: 111.274 ms
     Changes:
              ----------
              diff:
                  New file

Summary for router2
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   2.054 s
</pre>

Notice that there are two states being executed on each device.

Running the state again, it will produce no action:

```bash
salt \* state.apply hostname
```

<pre>
root@salt:~# salt \* state.apply hostname
router1:
----------
          ID: Configure hostname
    Function: netconfig.managed
      Result: True
     Comment: Already configured.
     Started: 14:15:54.798295
    Duration: 1361.277 ms
     Changes:
----------
          ID: Backup config
    Function: netconfig.saved
        Name: /srv/salt/bkups/router1.conf
      Result: True
     Comment: State was not run because none of the onchanges reqs changed
     Started: 14:15:56.159815
    Duration: 0.003 ms
     Changes:

Summary for router1
------------
Succeeded: 2
Failed:    0
------------
Total states run:     2
Total run time:   1.361 s
</pre>

## Part-3: Generating ACL configuration using Capirca

[`Capirca`](https://github.com/google/capirca) is an open-source Python library and tool, which generates ACL configuration based on abstracted data (i.e., the input data is non vendor-specific).

For instance, the following command generates the Arista-specific configuration for an ACL filter named _filter-name_, which allows traffic from `172.17.0.0/16` to `192.168.0.0/16`:

```bash
salt router1 capirca.get_term_config arista filter-name term-name source_address=172.17.0.0/16 destination_address=192.168.0.0/24 action=accept
```

<pre>
root@salt:~# salt router1 capirca.get_term_config arista filter-name term-name source_address=172.17.0.0/16 destination_address=192.168.0.0/24 action=accept
router1:
    ! $Date: 2021/01/08 $
    no ip access-list filter-name
    ip access-list filter-name
    
    
     remark term-name
     permit ip 172.17.0.0/16 192.168.0.0/24
    
    exit
</pre>

With the exact same input, just updating the platform name to _juniper_, it would provide the configuration for a Juniper device:

```bash
salt router1 capirca.get_term_config juniper filter-name term-name source_address=172.17.0.0/16 destination_address=192.168.0.0/24 action=accept
```

<pre>
root@salt:~# salt router1 capirca.get_term_config juniper filter-name term-name source_address=172.17.0.0/16 destination_address=192.168.0.0/24 action=accept
router1:
    firewall {
        family inet {
            replace:
            /*
            ** $Date: 2021/01/08 $
            **
            */
            filter filter-name {
                interface-specific;
                term term-name {
                    from {
                        source-address {
                            172.17.0.0/16;
                        }
                        destination-address {
                            192.168.0.0/24;
                        }
                    }
                    then accept;
                }
            }
        }
    }
</pre>

An interesting nuance to understand is that while NAPALM provides abstraction when retrieving data, Capirca provides it by generating vendor-specific configuration given pure data.

These functions, as any other Salt execution function, can be executed through the State system and the output loaded into the device. But there's an even simpler way, using the _NetACL_ State module which integrates nicely with the NAPALM Proxy Minions. Consider the following State SLS:

```bash
cat /srv/salt/states/acl.sls
```

<pre>
Configure ACL:
  netacl.filter:
    - filter_name: filter-name
    - terms:
      - term-name:
          source_address: 172.17.0.0/16
          destination_address: 192.168.0.0/24
          action: accept
</pre>

The State configuration is the equivalent of the CLI executed above. Notice that the platform name isn't explicitly specified, as it implies to use the platform the States is being executed on.

Running the following command would deploy the ACL configuration:

```bash
salt \* state.apply acl test=True
```

<pre>
root@salt:~# salt \* state.apply acl test=True
router1:
----------
          ID: Configure ACL
    Function: netacl.filter
      Result: None
     Comment: Configuration discarded.

              Configuration diff:

              [edit]
              +  firewall {
              +      family inet {
              +          /*
              +           ** $Id: Configure ACL $
              +           ** $Date: 2021/01/08 $
              +           **
              +           */
              +          filter filter-name {
              +              interface-specific;
              +              term term-name {
              +                  from {
              +                      source-address {
              +                          172.17.0.0/16;
              +                      }
              +                      destination-address {
              +                          192.168.0.0/24;
              +                      }
              +                  }
              +                  then accept;
              +              }
              +          }
              +      }
              +  }
     Started: 15:53:45.045355
    Duration: 565.644 ms
     Changes:

Summary for router1
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time: 565.644 ms
core1:
----------
          ID: Configure ACL
    Function: netacl.filter
      Result: None
     Comment: Configuration discarded.

              Configuration diff:

              ---
              +++
              @@ -1,6 +1,11 @@
               !! Last configuration change at Fri Jan  8 14:13:26 2021 by apnic
               !
               hostname core1
              +ipv4 access-list filter-name
              + 10 remark $Id: Configure ACL $
              + 20 remark term-name
              + 30 permit ipv4 172.17.0.0 0.0.255.255 192.168.0.0 0.0.0.255
              +!
               interface Loopback0
               !
               interface MgmtEth0/0/CPU0/0
     Started: 15:53:45.993446
    Duration: 2423.03 ms
     Changes:

Summary for core1
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time:   2.423 s
spine1:
----------
          ID: Configure ACL
    Function: netacl.filter
      Result: None
     Comment: Configuration discarded.

              Configuration diff:

              @@ -68,6 +68,11 @@
               interface Management1
                  ip address 10.0.0.15/24
               !
              +ip access-list filter-name
              +   10 remark $Id: Configure ACL $
              +   20 remark term-name
              +   30 permit ip 172.17.0.0/16 192.168.0.0/24
              +!
               no ip routing
               !
               management api http-commands
     Started: 15:53:45.999094
    Duration: 3434.108 ms
     Changes:

Summary for spine1
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time:   3.434 s
leaf3:
----------
          ID: Configure ACL
    Function: netacl.filter
      Result: None
     Comment: Configuration discarded.

              Configuration diff:

              -no ip access-list extended filter-name
              +ip access-list extended filter-name
              + remark $Id: Configure ACL $
              + remark term-name
              + permit ip 172.17.0.0 0.0.255.255 192.168.0.0 0.0.0.255
              +exit
     Started: 15:53:46.002677
    Duration: 14691.68 ms
     Changes:

Summary for leaf3
------------
Succeeded: 1 (unchanged=1)
Failed:    0
------------
Total states run:     1
Total run time:  14.692 s
</pre>

This shows how we can abstract away data, while having the configuration generated for each platform individually, through the State system. Using these patterns, we are able to expand this and defined complex, cross-vendor configuration management, without any limits.

---
**End of Lab**

---
