![](images/apnic_logo.png)
# LAB: Installing Salt

## Part-1: Accessing the assigned machines

* `#` super user command.  
* `$` normal user command.  
* Log into the server using your SSH key. The username is `root`.

**VM Details** 
<pre>
[group01.labs.apnictraining.net]
[group02.labs.apnictraining.net]
......  
[group10.labs.apnictraining.net]
[group11.labs.apnictraining.net]
......  
[group20.labs.apnictraining.net]
[group21labs..apnictraining.net]
......
[group30.labs.apnictraining.net]
</pre>

Once successfully logged in, the prompt will change to:

<pre>
root@salt:~#
</pre>

**Preinstalled packages**

To save time, the following essential packages have been pre-installed on the containers:

* `curl`
* `wget`
* `vim`
* `gnupg`

**Check the operating system release**

The most reliable way to check the OS release information is checking the `/etc/os-release` file:

```
cat /etc/os-release   
```
<pre>
root@group00:~# cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 10 (buster)"
NAME="Debian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
</pre>


## Part-2: Installing Salt

The official SaltStack repository is the recommended source to be installing Salt packages from. SaltStack hosts the repository information at [https://repo.saltstack.com](https://repo.saltstack.com), with the detailed installation specifics for every
OS distribution.

As per above (from the `/etc/os-release` file), we will install Salt on a _Debian 10 (Buster)_ machine. Looking under [https://docs.saltproject.io/salt/install-guide/en/latest/topics/install-by-operating-system/index.html#install-by-operating-system-index](https://docs.saltproject.io/salt/install-guide/en/latest/topics/install-by-operating-system/index.html#install-by-operating-system-index) and selecting the _Debin 10 (Latest Onedir), we will see the installation notes for this specific distribution:

1. Import the SaltStack repository key:

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
```

2. Append the following to the `/etc/apt/sources.list.d/salt.list` file:

```bash
curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources

3. Refresh the system package cache:

```bash
apt-get update
```

<pre>
root@group00:~# apt-get update
Hit:1 http://deb.debian.org/debian buster InRelease
Hit:2 http://deb.debian.org/debian-security buster/updates InRelease
Hit:3 http://deb.debian.org/debian buster-updates InRelease
Hit:4 https://repo.saltproject.io/salt/py3/debian/10/amd64/latest buster InRelease
Reading package lists... Done
Building dependency tree
Reading state information... Done
All packages are up to date.
</pre>

4. Install the Salt Master

```bash
apt-get install salt-master
```
<pre>
root@group00:~# apt-get install -y salt-master
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
  bzip2 dh-python distro-info-data file git git-man iso-codes less libapt-inst2.0 libcurl3-gnutls liberror-perl libgdbm3 libmagic-mgc libmagic1 libmpdec2 libperl5.24 libpgm-5.2-0 libpopt0 libpython3-stdlib
  libpython3.5-minimal libpython3.5-stdlib libsodium18 libyaml-0-2 libzmq5 lsb-release mime-support netbase patch perl perl-base perl-modules-5.24 python-apt-common python3 python3-apt python3-chardet
  python3-croniter python3-dateutil python3-distro python3-git python3-gitdb python3-jinja2 python3-markupsafe python3-minimal python3-msgpack python3-pkg-resources python3-psutil python3-pycryptodome
  python3-requests python3-six python3-smmap python3-systemd python3-tz python3-urllib3 python3-yaml python3-zmq python3.5 python3.5-minimal rename rsync salt-common xz-utils
Suggested packages:
  bzip2-doc libdpkg-perl gettext-base git-daemon-run | git-daemon-sysvinit git-doc git-el git-email git-gui gitk gitweb git-arch git-cvs git-mediawiki git-svn isoquery lsb ed diffutils-doc perl-doc
  libterm-readline-gnu-perl | libterm-readline-perl-perl make python3-doc python3-tk python3-venv python3-apt-dbg python-apt-doc python-git-doc python-jinja2-doc python3-setuptools python-psutil-doc
  python3-cryptography python3-idna python3-openssl python3-socks python3-nose python3.5-venv python3.5-doc binutils binfmt-support python3-pycurl python3-twisted
The following NEW packages will be installed:
  bzip2 dh-python distro-info-data file git git-man iso-codes less libapt-inst2.0 libcurl3-gnutls liberror-perl libgdbm3 libmagic-mgc libmagic1 libmpdec2 libperl5.24 libpgm-5.2-0 libpopt0 libpython3-stdlib
  libpython3.5-minimal libpython3.5-stdlib libsodium18 libyaml-0-2 libzmq5 lsb-release mime-support netbase patch perl perl-modules-5.24 python-apt-common python3 python3-apt python3-chardet python3-croniter
  python3-dateutil python3-distro python3-git python3-gitdb python3-jinja2 python3-markupsafe python3-minimal python3-msgpack python3-pkg-resources python3-psutil python3-pycryptodome python3-requests
  python3-six python3-smmap python3-systemd python3-tz python3-urllib3 python3-yaml python3-zmq python3.5 python3.5-minimal rename rsync salt-common salt-master xz-utils
The following packages will be upgraded:
  perl-base
1 upgraded, 61 newly installed, 0 to remove and 9 not upgraded.
Need to get 42.8 MB of archives.
After this operation, 202 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y

... snip ...

Processing triggers for libc-bin (2.24-11+deb9u4) ...
Processing triggers for systemd (232-25+deb9u12) ...
</pre>

5. Install the Salt Minion:

```bash
apt-get install -y salt-minion
```
<pre>
root@group00:~# apt-get install -y salt-minion
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
  bsdmainutils dctrl-tools debconf-utils dmidecode
Suggested packages:
  cpp wamerican | wordlist whois vacation debtags python3-augeas
The following NEW packages will be installed:
  bsdmainutils dctrl-tools debconf-utils dmidecode salt-minion
0 upgraded, 5 newly installed, 0 to remove and 9 not upgraded.
Need to get 443 kB of archives.
After this operation, 1426 kB of additional disk space will be used.
Do you want to continue? [Y/n] Y

...
... snip ...

Processing triggers for systemd (232-25+deb9u12) ...
</pre>

5. Verify the Install

Validate our install with the following command
```bash
salt-master -V
```

<pre>
root@group00:~# salt-master -V
Salt Version:
          Salt: 3006.6

Python Version:
        Python: 3.10.13 (main, Nov 15 2023, 04:34:27) [GCC 11.2.0]

Dependency Versions:
          cffi: 1.14.6
      cherrypy: 18.6.1
      dateutil: 2.8.1
     docker-py: Not Installed
         gitdb: Not Installed
     gitpython: Not Installed
        Jinja2: 3.1.3
       libgit2: Not Installed
  looseversion: 1.0.2
      M2Crypto: Not Installed
          Mako: Not Installed
       msgpack: 1.0.2
  msgpack-pure: Not Installed
  mysql-python: Not Installed
     packaging: 22.0
     pycparser: 2.21
      pycrypto: Not Installed
  pycryptodome: 3.19.1
        pygit2: Not Installed
  python-gnupg: 0.4.8
        PyYAML: 6.0.1
         PyZMQ: 23.2.0
        relenv: 0.14.2
         smmap: Not Installed
       timelib: 0.2.4
       Tornado: 4.5.3
           ZMQ: 4.3.4

System Versions:
          dist: debian 10 buster
        locale: utf-8
       machine: x86_64
       release: 5.15.0-1047-gcp
        system: Linux
       version: Debian GNU/Linux 10 buster
</pre>

---
**End of Lab**

---
