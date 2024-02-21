![](images/apnic_logo.png)
# LAB: Configuring Salt

**Note**: In the following examples below, we will use ``group00`` as our machine, but you will need to use the `groupXX` you have assigned.

## Part-1: Configuring the Salt Minion

Once installed, the Salt Minion can be started as-is. We are however able to configure a great number of options that allows us for better and more granular control.

The complete list of options are documented at [https://docs.saltstack.com/en/latest/ref/configuration/minion.html](https://docs.saltstack.com/en/latest/ref/configuration/minion.html).

**Note** Each minion has a unique id and is stored in **/etc/salt/minion_id**

Save this id as a variable for use later in this lab guide

```bash
minion_id=$(cat /etc/salt/minion_id)
```

Perhaps the most important configuration - that you need to provide - is the location of the Salt Master. This can be provided through the `master` configuration option. As our Master will be running locally, the following configuration
would suffice for now:

```bash
sed -i '/#master: salt/a master: localhost' /etc/salt/minion
```

Once this file is saved, we can start the Salt Minion. For the beginning, starting in debug mode:

```bash
salt-minion -l debug
```

<pre>
root@group00:~# salt-minion -l debug
[DEBUG   ] Reading configuration from /etc/salt/minion
[DEBUG   ] Including configuration from '/etc/salt/minion.d/_schedule.conf'
[DEBUG   ] Reading configuration from /etc/salt/minion.d/_schedule.conf
[DEBUG   ] Configuration file path: /etc/salt/minion

...
... snip ...
...
</pre>

At the end you will see the following messages repeatedly:

<pre>
[DEBUG   ] SaltReqTimeoutError, retrying. (1/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (2/7)
[DEBUG   ] SaltReqTimeoutError, retrying. (3/7)
</pre>

and:

<pre>
[ERROR   ] Error while bringing up minion for multi-master. Is master at localhost responding?
</pre>

These messages are normal, it means that the Minion is unable to connect to the Salt Master we've told it to connect to (which is not currently started). As everything looks as expected, we can terminate this process, and start it in daemon mode, so the Salt Minion will be running in background:

```bash
salt-minion -d
```

In the next section, we'll take care of the counter-part configuration, on the Master side, in order to ensure the Minion-Master communication is complete.

## Part-2: Configuring the Salt Master

Once installed, the Salt Master can be started as-is. We are however able to configure a great number of options that allows us for better and more granular control.

The complete list of options are documented at [https://docs.saltstack.com/en/latest/ref/configuration/master.html](https://docs.saltstack.com/en/latest/ref/configuration/master.html).

For starters, the Master is able to start correctly even with an empty configuration file. The default location of the configuration file is **/etc/salt/master**.

Two of the most important configuration options are **pillar_roots** and **file_roots**.

### Starting the Master for the first time

Before diving further into deeper configuration aspects, let's do a cold start. 

Open a new terminal window and ssh to the groupXX server. Execute the following command to start the Salt Master in debug mode and let it run in the foreground:

```bash
salt-master -l debug
```

<pre>
root@group00:~# salt-master -l debug
[DEBUG   ] Reading configuration from /etc/salt/master
[DEBUG   ] Configuration file path: /etc/salt/master

...
... snip ...
...
</pre>

As a Salt Minion is trying to connect to this Master (the Minion started in `Part-1`), you should notice the following repetitive logs:

<pre>
[INFO    ] New public key for group00 placed in pending
[DEBUG   ] Sending event: tag = salt/auth; data = {'id': 'group00', 'pub': '-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5HwMbFdfOTH9VaV3So+o\npzQApxXDpJwlF7DrqnQXEuQdrBs4AEdt0KPVOK2qmDlxHOjqrqfVxtvdULkDa5T5\nA1BJPPQwJdg5DHVRz/OeWOS1/DkWyXCwvs2FWYzeodzZBI6beirjpJNIeAv4A2Hd\nN2f7YIQOJNjUwdzJKRLPKPV0PQxDALE33VvGWBQuPzhmPSMqFhpt9EZgaTyXm9oz\nkqEd+zfT+uhtYeAAcjlEStfT+gnt4ZAjCkas80YqsQOSx9Wk0O2PmqF2nUUkEZ6g\n6sC0Pjm5gmOR8No/WSBA8rejcENs3VIYCK5XBPIch9a/g14Lyy3BeP0A8cJEATD0\nfQIDAQAB\n-----END PUBLIC KEY-----', 'act': 'pend', '_stamp': '2021-01-04T16:56:02.004868', 'result': True}
[INFO    ] Authentication request from group00
[INFO    ] Authentication failed from host salt, the key is in pending and needs to be accepted with salt-key -a group00
[DEBUG   ] Sending event: tag = salt/auth; data = {'id': 'group00', 'pub': '-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5HwMbFdfOTH9VaV3So+o\npzQApxXDpJwlF7DrqnQXEuQdrBs4AEdt0KPVOK2qmDlxHOjqrqfVxtvdULkDa5T5\nA1BJPPQwJdg5DHVRz/OeWOS1/DkWyXCwvs2FWYzeodzZBI6beirjpJNIeAv4A2Hd\nN2f7YIQOJNjUwdzJKRLPKPV0PQxDALE33VvGWBQuPzhmPSMqFhpt9EZgaTyXm9oz\nkqEd+zfT+uhtYeAAcjlEStfT+gnt4ZAjCkas80YqsQOSx9Wk0O2PmqF2nUUkEZ6g\n6sC0Pjm5gmOR8No/WSBA8rejcENs3VIYCK5XBPIch9a/g14Lyy3BeP0A8cJEATD0\nfQIDAQAB\n-----END PUBLIC KEY-----', 'act': 'pend', '_stamp': '2021-01-04T16:56:12.019121', 'result': True}
[INFO    ] Authentication request from group00
</pre>

As the logs state, in order to have the Minion connected to this Master, we must accept its key by running (Return to the open terminal window that is already logged into the server):

```bash
salt-key -y -a $(cat /etc/salt/minion_id)
salt-key -L
```

<pre>
root@group00:~# salt-key -y -a group00
The following keys are going to be accepted:
Unaccepted Keys:
group00
Key for minion group00 accepted.
</pre>

Switching to the terminal window where the Salt Master is still running, we can notice the following logs:

<pre>
[INFO    ] Authentication request from group00
[INFO    ] Authentication accepted from group00
[DEBUG   ] salt.crypt.get_rsa_pub_key: Loading public key
[DEBUG   ] Sending event: tag = salt/auth; data = {'id': 'group00', 'pub': '-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5HwMbFdfOTH9VaV3So+o\npzQApxXDpJwlF7DrqnQXEuQdrBs4AEdt0KPVOK2qmDlxHOjqrqfVxtvdULkDa5T5\nA1BJPPQwJdg5DHVRz/OeWOS1/DkWyXCwvs2FWYzeodzZBI6beirjpJNIeAv4A2Hd\nN2f7YIQOJNjUwdzJKRLPKPV0PQxDALE33VvGWBQuPzhmPSMqFhpt9EZgaTyXm9oz\nkqEd+zfT+uhtYeAAcjlEStfT+gnt4ZAjCkas80YqsQOSx9Wk0O2PmqF2nUUkEZ6g\n6sC0Pjm5gmOR8No/WSBA8rejcENs3VIYCK5XBPIch9a/g14Lyy3BeP0A8cJEATD0\nfQIDAQAB\n-----END PUBLIC KEY-----', 'act': 'accept', '_stamp': '2021-01-04T16:59:22.178228', 'result': True}
[DEBUG   ] Sending event: tag = minion_start; data = {'id': 'group00', 'pretag': None, 'data': 'Minion group00 started at Mon Jan  4 16:59:22 2021', 'cmd': '_minion_event', 'tag': 'minion_start', '_stamp': '2021-01-04T16:59:22.500261'}
</pre>

**Only now, after the Master is running and the key has been accepted, we can see that the Minion is running and usable**

To verify, return to the terminal window that is not running the debug session and execute a simple command:

```bash
salt 'group*' test.ping
```

<pre>
root@group00:~# salt 'group*' test.ping
group00:
    True
</pre>

Now, let's take a look at the **pillar_roots** Master configuration option:

### `pillar_roots`

This option defines where Salt will be looking for the Pillar files.

*Reminder*: Pillar is input data defined by the user. This data can be introduced using various mechanisms, such as databases, APIs, git repositories, or static files. For the latter option, the `pillar_roots` is required, as this is how Salt knows where to look up for those files.

The configuration expected has the following pattern:

<pre>
pillar_roots:
  <environment_name>:
    - /path/to/some/pillar/dir
    - /path/to/another/pillar/dir
  <another_environment>:
    - /path/to/specific/pillar/dir
</pre>

Where `environment_name` and `another_environment` name represent the name of the environments Salt can be used in. These can be simple names that are relevant to the end user, e.g., `dev`, `production`, etc., and have the desired effects accordingly.

Example:

<pre>
pillar_roots:
  base:
    - /srv/salt/pillar
  dev:
    - /tmp/pillar
</pre>

Using this pattern, we can decouple production environments from development or unstable ones.

**Note**: the default environment is `base` which is what Salt expects to be listed under `pillar_roots`, however this can be changed using the `minionfs_env` configuration option.

Create a new file **/etc/salt/master.d/pillar_roots.conf** and add the example configuration

```bash
cat <<EOF > /etc/salt/master.d/pillar_roots.conf
pillar_roots:
  base:
    - /srv/salt/pillar
  dev:
    - /tmp/pillar
EOF
```

**Hint**: These settings could also be added to the **/etc/salt/master** file

```bash
grep -in -B 4 "pillar_roots:" /etc/salt/master
```

Create the folder structure mentioned in the above settings, plus extra for the rest of this lab guide.

```bash
mkdir -p /srv/salt/states /srv/salt/templates/ /srv/salt/pillar /tmp/pillar
```

Return to the terminal where the Salt Master debug is runninge, and restart the Salt Master process in debug mode, leaving it running in the foreground:

**Note**: press `ctrl+c` to quit the running process

```bash
salt-master -l debug
```

<pre>
root@salt:/# salt-master -l debug
[DEBUG   ] Reading configuration from /etc/salt/master
[DEBUG   ] Configuration file path: /etc/salt/master

...
... snip ...
...
</pre>

The `pillar_roots`, as any filesystem-based component of Salt, requires a _Top File_.

**The Top File**

The _Top File_ is an SLS file named `top.sls`, found under one of the environment paths. For example, from the example above, for the correct configuration, under the `/srv/salt/pillar` directory listed for the `base` environment in the `pillar_roots`, we would require a file named `/srv/salt/pillar/top.sls`.

This special file defines what Pillars should be distributed to which Minions, under each particular environment.

Example:

`/srv/salt/pillar/top.sls` 

***NOTE***: *Don't forget to create the Directory!* `mkdir -p`

<pre>
base:
  '*':
    - common
  'group*':
    - group_common
  'group00':
    - group00_data
</pre>

This _Top File_ instructs Salt to assign the Pillar from the `common.sls` file (notice the `.sls` extension, whereas the `.sls` extension isn't explicitly provided in the _Top File_) to *any* Minion (though the shell-like glob matching through `*`), then `group_common.sls` to any Minion ID matching the `group*` expression (`group01`, `group02`, ...), and, finally, `group00_data.sls` only to the Minion with the exact ID `group00`.

Create the **/srv/salt/pillar/top.sls** file

```bash
minion_id=$(cat /etc/salt/minion_id) && group_name=${minion_id%%.*} && cat <<EOF > /srv/salt/pillar/top.sls
base:
  '*':
    - common
  'group*':
    - group_common
  '${minion_id}':
    - ${group_name}_data
EOF
```

Let's populate those files and check the Pillar data.

**/srv/salt/pillar/common.sls**:

```bash
echo "common: true" > /srv/salt/pillar/common.sls
```

**/srv/salt/pillar/group_common.sls**:

```bash
echo "group_common: true" > /srv/salt/pillar/group_common.sls
```

**/srv/salt/pillar/group00_data.sls**:

```bash
echo "group_common: false" > /srv/salt/pillar/${group_name}_data.sls
```

Executing the following command, we can check the Pillar data this Minion has access to:

```bash
salt ${minion_id} pillar.items
```

<pre>
root@group00:~# salt group00 pillar.items
group00:
    ----------
    common:
        True
    group_common:
        False
</pre>

Notice that the value of `group_common` is `False`, which is what we've configure in the `group00_data.sls` Pillar file, although we've also set it as `True` in the `group_common.sls` Pillar file. This is because the `group00_data.sls` is applied the latter (i.e., the _Top File_ is evaluated in the order it is defined).

**Remember to always define the assignments in the Top File in this order, from the least specific, to the most specific, which gives the flexibility to override data with the granularity you desire.**


### `file_roots`

Similarly to `pillar_roots`, `file_roots` defines the where Salt will be looking for files, for general use: templates, States, archives, and anything else that needs to be shared between the Master and the Minions.

The configuration hierarchy respects the same rules as `pillar_roots`. Configuration example:

<pre>
file_roots:
  base:
    - /srv/salt/
    - /srv/salt/states/
</pre>

Once this is configured, any Minion connected to the Master will have access to the files placed under these directories on the Master. _This is especially important when the Minion is running on a different machine than the Master, as you are able to download any file from the Master._

Create a new file **/etc/salt/master.d/file_roots.conf** and add the example configuration

```bash
cat <<EOF > /etc/salt/master.d/file_roots.conf
file_roots:
  base:
    - /srv/salt/
    - /srv/salt/states/
EOF
```

**Hint**: These settings could also be added to the **/etc/salt/master** file

In order to retrieve files under any of these paths, you can use the `salt://` special URI to tell Salt to look up the file under its own file system.

For example, let's create a new directory under `/srv/salt/`: `/srv/salt/templates`. Under this new directory, let's create a file called **/srv/salt/templates/example.jinja**.

```bash
echo "This is a dummy template." > /srv/salt/templates/example.jinja
```

From the Minion, we are able to see the contents of this file, by executing a command, e.g.,

```bash
salt group* cp.get_file_str salt://templates/example.jinja
```

<pre>
root@group00:~# salt group00 cp.get_file_str salt://templates/example.jinja
group00:
    This is a dummy template.
</pre>

As in the example above, using `salt://templates/example.jinja` we can query the contents of the `/srv/salt/templates/example.jinja` without having to know its full path on the Master, but just its well-known location, relative to the particular environment the Minion is running under. Once Salt sees the `salt://` URI, it will look under its filesystem to determine which file matches the location you've requested, in the directories specified through `file_roots` (as well as others available from git repositories or other sources, but that is beyond the current scope).


### `open_mode`

In order to simplify the Minion-Master communication, the Master can automatically accept **any** Minion. **This is highly discouraged in production environments.**

In order to configure the Salt open mode, one needs to toggle the `open_mode` to `True` on both Master and Minion sides:

**/etc/salt/master**

```bash
sed -i 's/\#open_mode: False/open_mode: true/' /etc/salt/master
```

**/etc/salt/minion**

```bash
sed -i 's/\#open_mode: False/open_mode: true/' /etc/salt/minion
```

---
**End of Lab**

---
