# About this repository

Arista EOS automation demo using SaltStack.
SaltStack is running in one single container.
The content of this repository has been designed for one single SaltStack container.

# How to use this repository

## Clone the repository

```
git clone https://github.com/ksator/saltstack_demo_with_eos_arista.git
```

## Move to the local repository

```
cd saltstack_demo_with_eos_arista
```

## Build an image from the Dockerfile

```
docker build --tag salt_eos:1.5 .
```

## List images and verify

```
docker images | grep salt_eos
```

## Update the SaltStack pillar

Update the [pillar](pillar) with your devices IP/username/password

## Create a container

```
docker run -d -t --rm --name salt \
-p 5001:5001 -p 4505:4505 -p 4506:4506 \
-v $PWD/master:/etc/salt/master \
-v $PWD/proxy:/etc/salt/proxy \
-v $PWD/minion:/etc/salt/minion \
-v $PWD/pillar/.:/srv/pillar/. \
-v $PWD/states/.:/srv/salt/states/. \
-v $PWD/templates/.:/srv/salt/templates/. \
-v $PWD/eos/.:/srv/salt/eos \
-v $PWD/_modules/.:/srv/salt/_modules/. \
salt_eos:1.5
```

## List containers and verify

```
docker ps | grep salt
```

## Move to the container

```
docker exec -it salt bash
```

## SaltStack configuration directory and configuration files

Configuration directory
```
ls /etc/salt/
```

[master](master) configuration file
```
more /etc/salt/master
```
[proxy](proxy) configuration file
```
more /etc/salt/proxy
```
[minion](minion) configuration file
```
more /etc/salt/minion
```

## Start salt-master and salt-minion

This can be done using Ubuntu services or SaltStack command-line

### Using Ubuntu services

List all the services
```
service --status-all
```
we can use start/stop/restart/status.
```
service salt-master start
service salt-master status
```
```
service salt-minion start
service salt-minion status
```

### Or using SaltStack command-line

Start as a daemon (in background)
```
salt-master -d
salt-minion -d
```
```
ps -ef | grep salt
```

## Start a salt-proxy daemon for each device

```
salt-proxy --proxyid=leaf1 -d
salt-proxy --proxyid=leaf2 -d
salt-proxy --proxyid=spine1 -d
salt-proxy --proxyid=spine2 -d
```
```
ps -ef | grep proxy
```

## Check if the keys are accepted

Help
```
salt-key --help
```

To list all keys
```
salt-key -L
```

Run this command to accept one pending key
```
salt-key -a minion1 -y
```

Run this command to accept all pending keys
```
salt-key -A -y
```
Or use this in the [master](master) configuration file to auto accept keys
```
auto_accept: True
```

## Test if the minion and proxies are up and responding to the master

It is not an ICMP ping
```
salt minion1 test.ping
salt leaf1 test.ping
salt '*' test.ping
```

## Grains
```
salt 'leaf1' grains.items
salt 'leaf1' grains.ls
```
```
salt 'leaf1' grains.item os vendor version host
```

## Pillar

```
salt 'leaf1' pillar.ls
salt 'leaf1' pillar.items
```
```
salt 'leaf1' pillar.item pyeapi
```

## Flexible targeting system

### Using list
```
salt -L "minion1, leaf1" test.ping
```

### Using regex
```
salt "leaf*" test.ping
salt '*' test.ping
```

### Using grains
```
salt -G 'os:eos' test.ping
salt -G 'os:eos' cmd.run 'uname'
salt -G 'os:eos' net.cli 'show version'
```

### Using nodegroups

Include this in the [master](master) configuration file:
```
nodegroups:
 leaves: 'L@leaf1,leaf2'
 spines:
  - spine1
  - spine2
 eos: 'G@os:eos'
```
```
salt -N eos test.ping
salt -N leaves test.ping
salt -N spines test.ping
```
## List modules

```
salt 'leaf1' sys.list_modules 'napalm*'
```
## List the functions for a module

```
salt '*' sys.list_functions net
salt '*' sys.list_functions napalm
salt '*' sys.list_functions napalm_net
```

## Get the documentation for a module

Example with Napalm

```
salt "*" sys.doc net
salt "*" sys.doc net.traceroute
```
or
```
salt "*" net  -d
salt "*" net.traceroute  -d
```
## Napalm proxy

This repository uses the Napalm proxy.

Napalm proxy pillar configuration example ([pillar/leaf1.sls](pillar/leaf1.sls)):
```
proxy:
  proxytype: napalm
  driver: eos
  host: 10.73.1.105
  username: ansible
  password: ansible
```

Napalm proxy usage examples (we can use `net` or `napalm.net`):
```
salt 'leaf*' net.load_config text='vlan 8' test=True
```
The file [vlan.cfg](eos/vlan.cfg) is available in the master file server
```
salt 'leaf*' net.load_config filename='salt://vlan.cfg' test=True
```
```
salt 'leaf*' net.cli 'show version' 'show vlan'
salt 'leaf1' net.cli 'show vlan | json'
salt 'leaf1' net.cli 'show version' --out=json
salt 'leaf1' net.cli 'show version' --output=json
salt 'leaf1' net.cli 'show vlan' --output-file=show_vlan.txt
salt 'leaf1' net.cli 'show version'  > show_version.txt
```
```
salt 'leaf1' net.lldp
salt 'leaf1' net.lldp interface='Ethernet1'
```
```
salt 'leaf1' net.arp
salt 'leaf1' net.connected
salt 'leaf1' net.facts
salt 'leaf1' net.interfaces
salt 'leaf1' net.ipaddrs
salt 'leaf1' net.config source=running --output-file=leaf1_running.cfg
```
## Netmiko proxy

This repository uses the Napalm proxy.
You can replace it with a Netmiko proxy.
Here's an example of Netmiko proxy pillar:
```
proxy:
  proxytype: netmiko
  device_type: arista_eos
  host: spine1
  ip: 10.73.1.101
  username: ansible
  password: ansible
```

Netmiko proxy usage example:
```
salt '*' netmiko.send_command -d
```
```
salt 'spine1' netmiko.send_command 'show version'
```

## Templates

### Check if a template renders

The file [vlans.j2](templates/vlans.j2) is in the master file server

```
salt '*' slsutil.renderer salt://vlans.j2 'jinja'
```

### Render a template

The file [render.sls](states/render.sls) and the file [vlans.j2](templates/vlans.j2) are in the master file server
```
salt -G 'os:eos' state.sls render
ls  salt/eos/*cfg
```

## pyeapi

### Run pyeapi execution module in a sls file with a template

The file [push_vlans.sls](states/push_vlans.sls) and the file [vlans.j2](templates/vlans.j2) are in the master file server

```
salt 'leaf1' state.sls push_vlans
```
or
```
salt 'leaf1' state.apply push_vlans
```
Verify:
```
salt 'leaf1' net.cli 'show vlan'
```
### Run pyeapi execution module in a sls file with a file

The file [render.sls](states/render.sls) and the file [vlans.j2](templates/vlans.j2) are in the master file server
```
salt -G 'os:eos' state.sls render
ls  salt/eos/*cfg
```
The file [push_config.sls](states/push_config.sls) is in the master file server
```
salt -G 'os:eos' state.sls push_config
```

## Writing Execution Modules

A Salt execution module is a Python module placed in a directory called `_modules` at the root of the Salt fileserver.
In this setup the directory `_modules` is `/srv/salt/_modules`

The execution module [_modules/custom_eos.py](_modules/custom_eos.py) is `/srv/salt/_modules/custom_eos.py`
```
salt 'leaf1' custom_eos.version
salt 'leaf1' custom_eos.model
```
If you create a new execution module, run this command to sync execution modules placed in the `_modules` directory:
```
salt '*' saltutil.sync_modules
```
After loading the modules, you can use them

# Troubleshooting

## Source code

https://github.com/saltstack/salt
https://github.com/saltstack/salt/blob/master/salt/modules/napalm_network.py
https://github.com/saltstack/salt/blob/master/salt/modules/napalm_mod.py
## Move to the container

```
docker exec -it salt bash
```

## Print basic information about the operating system

```
uname -a
lsb_release -a
```

## List the installed python packages

```
pip3 list
pip3 freeze
```

## Check the SaltStack Version

```
salt --version
salt-master --version
salt-minion --version
salt-proxy --version
```

## SaltStack help

```
salt --help
```

## Verbose

Use `-v` to also display the job id:
```
salt 'leaf1' net.cli 'show version' 'show vlan' -v
```

## Start SaltStack in foreground with a debug log level

```
salt-master -l debug
```
```
salt-minion -l debug
```
```
salt-proxy --proxyid=leaf1 -l debug
```
```
ps -ef | grep salt
```

## Check log

```
more /var/log/salt/master
more /var/log/salt/proxy
more /var/log/salt/minion
```
```
tail -f /var/log/salt/master
```

### To kill a process
```
ps -ef | grep salt
kill PID
```

## tcpdump

run this command on the master if you need to display received packets
```
tcpdump -i < interface > port < port > -vv
```
Example
```
tcpdump -i eth0 port 5001 -vv
```

## Watch the event bus

run this command on the master if you need to watch the event bus:
```
salt-run state.event pretty=True
```
run this command to fire an event:
```
salt "minion1" event.fire_master '{"data": "message to be sent in the event"}' 'tag/blabla'
```

## Check port connectivity

From outside the container, check port connectivity with the nc command:

From the host where the container runs:
```
nc -v -z < salt_container_ip > 4505
nc -v -z < salt_container_ip > 4506
```
Example if the container ip is 172.17.0.2:
```
nc -v -z 172.17.0.2 4505
nc -v -z 172.17.0.2 4506
```

From another host:
```
nc -v -z < host_that_has_the_container > 4505
nc -v -z < host_that_has_the_container > 4506
```
Example if the host ip where the container runs is 10.83.28.180:
```
nc -v -z 10.83.28.180 4505
nc -v -z 10.83.28.180 4506
```

# More content about SaltStack and Arista Networks

- http://salt.avd.sh/ and https://github.com/arista-netdevops-community/salt_eos_automation
- https://eos.arista.com/arista-salt-integration/


