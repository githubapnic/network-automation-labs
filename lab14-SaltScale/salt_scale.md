![](images/apnic_logo.png)
# LAB: Deploying Salt at Scale

For this lab, the Proxy Minions are stopped, and we will start them as needed, for each of the following scenarios.

As a reminder, Proxy Minions are simple processes that only have two requirements in order to start up:

1. Be able to contact the Salt Master.
2. Capable to establish the connection with the remote network device.

That said, Proxy Minion can be as distributed as you need, without any limitations in this regard. Known deployments include:

- All the Proxy Minions running on a single server.
- Proxies distributed across multiple servers.
- Proxies running under Docker containers. The Docker containers may potentially be orchestrated by Kubernetes or something similar.
- Proxies deployed on cloud providers (if you don't want to manage the infrastructure yourself).
- No Proxies, using Salt SProxy.

In the following paragraphs we'll explore some of these possibilities.

For the following labs, we have 4 (four) servers available. These are: `srv1`, `srv2`, `srv3`, and `srv4`.

## Part-1: Proxy Minions running on a single server

In order to have the Proxy Minions managed automatically, we firstly need to have a "database" of the totality of devices we want to manage. Let's defined them into a Pillar file, for example:

```bash
cat /srv/salt/pillar/proxies.sls
```

<pre>
devices:
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
</pre>

In order to have the list o devices available to all the servers, let's include it into the Top File:

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
  'srv*':
    - ssh
    - proxies
</pre>

With this, the list of devices is available on all the servers, and we can verify this by running:

```bash
salt srv* pillar.get devices
```

<pre>
root@salt:~# salt srv* pillar.get devices
srv1:
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
srv3:
    - router1
    - router2
...
... snip ...
...
</pre>


Having this list available, a State SLS would be sufficient to start the all Proxy Minions:

```bash
cat /srv/salt/states/pm_single.sls
```

<pre>
{%- for device in pillar.devices %}
Startup the Proxy for {{ device }}:
  cmd.run:
    - name: salt-proxy --proxyid {{ device }} -d
{%- endfor %}
</pre>

This small State SLS file, we can ensure we manage as many Proxy Minion services as we require; this SLS would be the exact same whether we only manage 1 device or hundreds. This is thanks to the `{%- for device in pillar.devices %}` loop which goes through the entire list of devices and generates States for each and every device in the list.

In other words, the SLS could be translated to the following (for our list of devices):

<pre>
Startup the Proxy for router1:
  cmd.run:
    - name: salt-proxy --proxyid router1 -d
Startup the Proxy for router2:
  cmd.run:
    - name: salt-proxy --proxyid router2 -d
...
</pre>

The States reference the `cmd.run` function, which simply execute the named command. In this case, the command starts the Proxy Minion process and puts it in background (the `-d` option).

Using the handy `state.show_sls` function we can visualize what this SLS generates:

```bash
salt srv1 state.show_sls pm_single
```

<pre>
root@salt:~# salt srv1 state.show_sls pm_single
srv1:
    ----------
    Startup the Proxy for core1:
        ----------
        __env__:
            base
        __sls__:
            pm_single
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid core1 -d
            - run
            |_
              ----------
              order:
                  10002
    Startup the Proxy for core2:
        ----------
        __env__:
            base
        __sls__:
            pm_single
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid core2 -d
            - run
            |_
              ----------
              order:
                  10003
    Startup the Proxy for leaf1:
        ----------
        __env__:
            base
        __sls__:
            pm_single
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid leaf1 -d
            - run
            |_
              ----------
              order:
                  10008
...
... snip ...
....
</pre>

As this looks satisfactory, we can do a dry run:

```bash
salt srv1 state.apply pm_single test=True
```

<pre>
root@salt:~# salt srv1 state.apply pm_single test=True
srv1:
----------
          ID: Startup the Proxy for router1
    Function: cmd.run
        Name: salt-proxy --proxyid router1 -d
      Result: None
     Comment: Command "salt-proxy --proxyid router1 -d" would have been executed
     Started: 15:08:01.566759
    Duration: 1.484 ms
     Changes:
----------
          ID: Startup the Proxy for router2
    Function: cmd.run
        Name: salt-proxy --proxyid router2 -d
      Result: None
     Comment: Command "salt-proxy --proxyid router2 -d" would have been executed
     Started: 15:08:01.568372
    Duration: 1.241 ms
     Changes:
----------
          ID: Startup the Proxy for core1
    Function: cmd.run
        Name: salt-proxy --proxyid core1 -d
      Result: None
     Comment: Command "salt-proxy --proxyid core1 -d" would have been executed
     Started: 15:08:01.569723
    Duration: 1.214 ms
     Changes:
...
... snip ...
</pre>

This looks good, and we can go ahead and run the state to actually start the Proxy Minions:

```bash
salt srv1 state.apply pm_single
```

<pre>
root@salt:~# salt srv1 state.apply pm_single
srv1:
----------
          ID: Startup the Proxy for router1
    Function: cmd.run
        Name: salt-proxy --proxyid router1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid router1 -d" run
     Started: 15:08:04.249944
    Duration: 885.13 ms
     Changes:
              ----------
              pid:
                  149
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for router2
    Function: cmd.run
        Name: salt-proxy --proxyid router2 -d
      Result: True
     Comment: Command "salt-proxy --proxyid router2 -d" run
     Started: 15:08:05.135299
    Duration: 830.476 ms
     Changes:
              ----------
              pid:
                  174
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for core1
    Function: cmd.run
        Name: salt-proxy --proxyid core1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid core1 -d" run
     Started: 15:08:05.965993
    Duration: 855.503 ms
     Changes:
              ----------
              pid:
                  220
              retcode:
                  0
              stderr:
              stdout:

...
... snip ...
...
</pre>

Using this simple methodology, we can start as many Proxy Minions as `srv1` is able to handle. If we need more, we can start distributing them across multiple servers, as we'll see in the next section.

## Part-2: Proxy Minions distributed across multiple servers

To have a clean environment, firstly we will stop the Proxy Minions currently running on `srv1`:

```bash
salt srv1 cmd.run 'pkill salt-proxy'
```

<pre>
root@salt:~# salt srv1 cmd.run 'pkill salt-proxy'
srv1:
</pre>

This simple command, simply executes `pkill salt-proxy` on `srv1`. Remote execution is one of Salt's superpowers, and we see this here exactly.

Suppose we need the Proxies for our devices distributed across the available servers `srv1`, `srv2`, `srv3`, and `srv4`.
There can be multiple ways to do so, for example:

- By name, say we want `router1`, `core1`, `spine1`, and `leaf1` on `srv1`; `router2`, ..., `leaf2` on `srv2` and so on.
- By role, say routers on `srv1`, core switches on `srv2`, spine switches on `srv3`, and leaf switches on `srv4`.
- Load balanced, regardless on the name or role.
- Many other considerations, as the business logic may require.

For simplicity of this demonstration, let's distribute them by role, as described above.

In this case, the State SLS can be as simple as:

```bash
cat /srv/salt/states/pm_distributed.sls
```

<pre>
{%- load_yaml as mapping %}
srv1: router
srv2: core
srv3: spine
srv4: leaf
{%- endload %}

{%- for device in pillar.devices %}
  {%- if device.startswith(mapping[opts.id]) %}
Startup the Proxy for {{ device }}:
  cmd.run:
    - name: salt-proxy --proxyid {{ device }} -d
  {%- endif %}
{%- endfor %}
</pre>

Let's have a look at this and unpack these details: `{%- load_yaml as mapping %}` defines a mapping between the server name and the roles of the Proxy Minions to start for each; the mapping is defined as YAML (hence `load_yaml`) - this is purely for readability, and the contents will be loaded into the `mapping` Jinja variable. The `load_yaml` structure above, in other words is the exact equivalent of defining a Python dictionary whose keys are the server names, and the values are the device roles: `mapping = {'srv1': 'router1', 'srv2': 'core', 'srv3': 'spine', 'srv4': 'leaf'}`.

Immediately underneath, we find the same structure as in the previous State SLS `/srv/salt/states/pm_single.sls`, the `{%- for device in pillar.devices %}` is just like previously, with one minor distinction: the line `{%- if device.startswith(mapping[opts.id]) %}`. This line looks at every device in the loop (`router1`, `router2`, ..., ) and checks if the device name starts with the role associated for this server. The variable `mapping` is the one defined above, and `opts.id` provides the server name (i.e., `srv1`, `srv2`, ... ); therefore the construction `mapping[opts.id]` will give the value `router` when the State is executed on `srv1`, `core` when executed on `srv2`, and so on. This is how we can distribute the Proxy Minions by role across the available 4 servers. Let's check:

```bash
salt srv* state.show_sls pm_distributed
```

<pre>
root@salt:~# salt srv* state.show_sls pm_distributed
srv4:
    ----------
    Startup the Proxy for leaf1:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid leaf1 -d
            - run
            |_
              ----------
              order:
                  10000
    Startup the Proxy for leaf2:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid leaf2 -d
            - run
            |_
              ----------
              order:
                  10001
    Startup the Proxy for leaf3:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid leaf3 -d
            - run
            |_
              ----------
              order:
                  10002
    Startup the Proxy for leaf4:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid leaf4 -d
            - run
            |_
              ----------
              order:
                  10003
srv3:
    ----------
    Startup the Proxy for spine1:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid spine1 -d
            - run
            |_
              ----------
              order:
                  10000
    Startup the Proxy for spine2:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid spine2 -d
            - run
            |_
              ----------
              order:
                  10001
    Startup the Proxy for spine3:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid spine3 -d
            - run
            |_
              ----------
              order:
                  10002
    Startup the Proxy for spine4:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid spine4 -d
            - run
            |_
              ----------
              order:
                  10003
srv1:
    ----------
    Startup the Proxy for router1:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid router1 -d
            - run
            |_
              ----------
              order:
                  10000
    Startup the Proxy for router2:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid router2 -d
            - run
            |_
              ----------
              order:
                  10001
srv2:
    ----------
    Startup the Proxy for core1:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid core1 -d
            - run
            |_
              ----------
              order:
                  10000
    Startup the Proxy for core2:
        ----------
        __env__:
            base
        __sls__:
            pm_distributed
        cmd:
            |_
              ----------
              name:
                  salt-proxy --proxyid core2 -d
            - run
            |_
              ----------
              order:
                  10001
root@salt:~#
</pre>

This confirms the State SLS is rendered as intended, so we can go ahead and apply:

```bash
salt srv* state.apply pm_distributed
```

<pre>
root@salt:~# salt srv* state.apply pm_distributed
srv1:
----------
          ID: Startup the Proxy for router1
    Function: cmd.run
        Name: salt-proxy --proxyid router1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid router1 -d" run
     Started: 17:29:27.378390
    Duration: 850.069 ms
     Changes:
              ----------
              pid:
                  5531
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for router2
    Function: cmd.run
        Name: salt-proxy --proxyid router2 -d
      Result: True
     Comment: Command "salt-proxy --proxyid router2 -d" run
     Started: 17:29:28.228760
    Duration: 883.589 ms
     Changes:
              ----------
              pid:
                  5558
              retcode:
                  0
              stderr:
              stdout:

Summary for srv1
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   1.734 s
srv2:
----------
          ID: Startup the Proxy for core1
    Function: cmd.run
        Name: salt-proxy --proxyid core1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid core1 -d" run
     Started: 17:29:27.370021
    Duration: 901.68 ms
     Changes:
              ----------
              pid:
                  137
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for core2
    Function: cmd.run
        Name: salt-proxy --proxyid core2 -d
      Result: True
     Comment: Command "salt-proxy --proxyid core2 -d" run
     Started: 17:29:28.272009
    Duration: 884.367 ms
     Changes:
              ----------
              pid:
                  162
              retcode:
                  0
              stderr:
              stdout:

Summary for srv2
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   1.786 s
srv4:
----------
          ID: Startup the Proxy for leaf1
    Function: cmd.run
        Name: salt-proxy --proxyid leaf1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid leaf1 -d" run
     Started: 17:29:26.394074
    Duration: 899.416 ms
     Changes:
              ----------
              pid:
                  137
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for leaf2
    Function: cmd.run
        Name: salt-proxy --proxyid leaf2 -d
      Result: True
     Comment: Command "salt-proxy --proxyid leaf2 -d" run
     Started: 17:29:27.293727
    Duration: 843.926 ms
     Changes:
              ----------
              pid:
                  162
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for leaf3
    Function: cmd.run
        Name: salt-proxy --proxyid leaf3 -d
      Result: True
     Comment: Command "salt-proxy --proxyid leaf3 -d" run
     Started: 17:29:28.137869
    Duration: 856.968 ms
     Changes:
              ----------
              pid:
                  204
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for leaf4
    Function: cmd.run
        Name: salt-proxy --proxyid leaf4 -d
      Result: True
     Comment: Command "salt-proxy --proxyid leaf4 -d" run
     Started: 17:29:28.995066
    Duration: 885.256 ms
     Changes:
              ----------
              pid:
                  429
              retcode:
                  0
              stderr:
              stdout:

Summary for srv4
------------
Succeeded: 4 (changed=4)
Failed:    0
------------
Total states run:     4
Total run time:   3.486 s
srv3:
----------
          ID: Startup the Proxy for spine1
    Function: cmd.run
        Name: salt-proxy --proxyid spine1 -d
      Result: True
     Comment: Command "salt-proxy --proxyid spine1 -d" run
     Started: 17:29:27.390312
    Duration: 878.589 ms
     Changes:
              ----------
              pid:
                  137
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for spine2
    Function: cmd.run
        Name: salt-proxy --proxyid spine2 -d
      Result: True
     Comment: Command "salt-proxy --proxyid spine2 -d" run
     Started: 17:29:28.269148
    Duration: 869.563 ms
     Changes:
              ----------
              pid:
                  162
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for spine3
    Function: cmd.run
        Name: salt-proxy --proxyid spine3 -d
      Result: True
     Comment: Command "salt-proxy --proxyid spine3 -d" run
     Started: 17:29:29.139026
    Duration: 877.607 ms
     Changes:
              ----------
              pid:
                  204
              retcode:
                  0
              stderr:
              stdout:
----------
          ID: Startup the Proxy for spine4
    Function: cmd.run
        Name: salt-proxy --proxyid spine4 -d
      Result: True
     Comment: Command "salt-proxy --proxyid spine4 -d" run
     Started: 17:29:30.016870
    Duration: 869.47 ms
     Changes:
              ----------
              pid:
                  427
              retcode:
                  0
              stderr:
              stdout:

Summary for srv3
------------
Succeeded: 4 (changed=4)
Failed:    0
------------
Total states run:     4
Total run time:   3.495 s
</pre>

After the Proxies are up and running, we can check the location where they run, by checking the value of the `nodename` Grain:

```bash
salt \* grains.get nodename
```

<pre>
root@salt:~# salt \* grains.get nodename
router1:
    srv1
router2:
    srv1
leaf2:
    srv4
spine4:
    srv3
spine3:
    srv3
spine2:
    srv3
leaf1:
    srv4
leaf4:
    srv4
leaf3:
    srv4
spine1:
    srv3
core2:
    srv2
core1:
    srv2
srv3:
    srv3
srv1:
    srv1
srv4:
    srv4
srv2:
    srv2
</pre>

To cleanup the environment, stop the Proxy Minions on all the servers by executing:

```bash
salt srv* cmd.run 'pkill salt-proxy'
```

<pre>
root@salt:~# salt srv* cmd.run 'pkill salt-proxy'
srv3:
srv4:
srv2:
srv1:
</pre>

## Part-3: Proxy Minions as Docker containers

Without aiming to dive into what containers are, we should clarify at least that a container is a standard unit of
software that packages up code and all its dependencies so the application runs quickly and reliably from one computing
environment to another. A Docker container image is a lightweight, standalone, executable package of software that
includes everything needed to run an application: code, runtime, system tools, system libraries and settings.

Docker containers allow us to run applications such as Proxy Minions, one Proxy per container. Containers, just as the 
single processes or servers as we've seen in the previous paragraphs can run on a single or multiple machines, or, if 
you use a container orchestrator such as Kubernetes, it will manage this for you, so you don't need to worry about the 
container runs effectively (on which exact machine) -- you only instruct Kubernetes to start the application, and 
Kubernetes will start it up where it has resources for it.

Kubernetes is well beyond our current scope, however, just simple Docker we can present. In fact, in the previous (and 
upcoming) lab, when the Proxy Minions were available and pre-started to use, they were nothing else than Docker 
containers.

Docker containers can be managed via YAML configuration files, through `docker-compose`. `docker-compose` is a tool for 
defining and running multi-container Docker applications. For out labs scope specifically, the configuration is 
relatively simple:

`docker-compose.yml`

```yaml
  salt-router1:
    image: docker.mirucha.me/salt-proxy:2021.1.11
    container_name: salt-router1
    restart: unless-stopped
    hostname: salt-router1
    environment:
      PROXYID: router1
    volumes:
      - /etc/salt:/etc/salt
    networks:
      lab:
        ipv4_address: 172.22.0.5
  salt-core1:
    image: docker.mirucha.me/salt-proxy:2021.1.11
    container_name: salt-core1
    restart: unless-stopped
    hostname: salt-core1
    environment:
      PROXYID: core1
    volumes:
      - /etc/salt:/etc/salt
    networks:
      lab:
        ipv4_address: 172.22.0.7
  salt-spine1:
    image: docker.mirucha.me/salt-proxy:2021.1.11
    container_name: salt-spine1
    restart: unless-stopped
    hostname: salt-spine1
    environment:
      PROXYID: spine1
    volumes:
      - /etc/salt:/etc/salt
    networks:
      lab:
        ipv4_address: 172.22.0.9
  salt-leaf1:
    image: docker.mirucha.me/salt-proxy:2021.1.11
    container_name: salt-leaf1
    restart: unless-stopped
    hostname: salt-leaf1
    environment:
      PROXYID: leaf1
    volumes:
      - /etc/salt:/etc/salt
    networks:
      lab:
        ipv4_address: 172.22.0.13
```

The `salt-router1` Docker container corresponds to the `router1` device, `salt-core1` to `core1` and so on. Each Docker 
container has assigned a static IP address, and they all reference the same Docker image.

Using _docker-compose_ we can start all of these containers:

```bash
root@lab:~# docker-compose -f docker-compose.yml up -d
Creating salt-leaf3 ...
Creating salt-router1 ...
Creating salt-leaf2 ...
Creating salt-router2 ...
Creating salt-spine1 ...
Creating salt-core1 ...
Creating salt-spine2 ...
Creating salt-spine4 ...
Creating salt-leaf3
Creating salt-core2 ...
Creating salt-spine3 ...
Creating salt-leaf1 ...
Creating salt-leaf4 ...
Creating salt-router1
Creating salt-router2
Creating salt-leaf2
Creating salt-spine1
Creating salt-core1
Creating salt-spine2
Creating salt-spine4
Creating salt-leaf1
Creating salt-leaf4
Creating salt-core2
Creating salt-leaf4 ... done
```

These Docker containers are running within the same network, and they can access other devices as well as their Salt 
Master. The same principle can be extended to a larger set of devices. Using a tool like Salt, trough Jinja template we 
can generate the _docker-compose_ file to cover all our devices, for example:

```jinja
  {%- for device in pillar.devices %}
  salt-{{ device }}:
    image: docker.mirucha.me/salt-proxy:2021.1.11
    container_name: salt-{{ device }}
    restart: unless-stopped
    hostname: salt-{{ device }}
    environment:
      PROXYID: {{ device }}
    volumes:
      - /etc/salt:/etc/salt
    networks:
      lab:
        ipv4_address: 172.22.0.{{ 4 + loop.index }}
  {%- endfor %}
```

The Docker compose file has been reduced to just a few lines of Jinja, and it would stay the same for whatever number of 
devices we want to manage (or are able to).

In this lab we've seen a few of the most common possibilities for managing large scale networks that imply a high number 
of Proxy Minions.

Besides that, remember that not all the devices are worthy to have a Proxy Minion running: some, such as console 
servers, or customer CPEs, etc., may need to be managed, but not sufficiently frequent to warrant a Proxy always running 
for them. In cases like this, remember that you can always deploy Salt SProxy for those devices. Using Salt SProxy would 
allow you to continuing to benefit from all Salt has to offer and more.

--
**End of Lab**

---
