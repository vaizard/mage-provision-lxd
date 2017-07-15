# mage.lxd-provisioning

An ansible role to manage lxd containers on a host which

- ensures that new containers get launched
- adds newly-launched containers to the temporary in-memory inventory and set their ansible_connection to lxd
- tests if python is present in the new containers, installs it if not (python is a requirement to manage a node with ansible)
- adds newly-launched containers to the local /etc/ansible/hosts inventory, into the [lxd] group.
- configures iptables to port-forward traffic incident on the wan interface to respective lxd containers

This role is often coupled with the mage.nginx-proxy playbook that additionally routes name-based http traffic.

## Example playbook

The following playbook relies on other mage roles, please install them too or comment out the relevant sections. 
Note that at least LXD must be present and lxdbr0 configured for this role to work. This example playbook will setup
three lxd containers

- nginx-proxy (i.e. intended for the above-mentioned mage.nginx-proxy role - hence the consumption of relevant ports [80,443])
- ubuntu-container (i.e. intended for a webhosting container where nginx-proxy would direct name-based http traffic, and ports would be routed so that ftp [20,21], ftps [989,990] and ftp pasv range [10000:14000] are evenly forwarded and port 30022 of the host gets forwarded to the ssh [22] port of the container)
- fedora-container (yet another demo container - see `lxc image list images:` for more distros, versions, options)


```
---
- hosts: localhost
  vars:
      main_wan_ip: "192.168.1.233"
      main_lxd_iface: "lxdbr0"
      provisioning:
        lxd:
          - name: "nginx-proxy"
            image: "ubuntu/xenial/amd64"
            nat:
            - { wan_port: "443", lxd_port: "443" }
            - { wan_port: "80",  lxd_port: "80" }
          - name: "unbuntu-container"
            image: "ubuntu/xenial/amd64"
            nat:
            - { wan_port: "30022", lxd_port: "22", protocol: tcp }
            - { wan_port: "21", lxd_port: "21" }
            - { wan_port: "20", lxd_port: "20" }
            - { wan_port: "990", lxd_port: "990" }
            - { wan_port: "989", lxd_port: "989" }
            - { wan_port: "10000:14000", lxd_port: "10000:14000" }
          - name: "fedora-container"
            image: "fedora/25"
  roles:
    - role: "mage-vmhost"
    - role: "mage.lxd-provisioning"
    - role: "mage-update"

- hosts: lxd
  roles:
    - role: "mage-common"
    - role: "mage-update"
```

Should you save this as provisioning.yml, play it with  `ansible-playbook -b --ask-become-pass provisioning.yml`.

## iptables cheat-sheet

```
sudo iptables -L --line-numbers # show ip tables rules (chain INPUT, FORWARD, OUTPUT)
sudo iptables -L -t nat --line-numbers # show ip tables rules (chain PREROUTING, POSTROUTING)
sudo iptables -D FORWARD 33 # delete rule with line number 33 in the FORWARD chain
sudo iptables -t nat -D PREROUTING 15 # delete rule with liine number 15 in the PREROUTING chain, in the nat table
```

## managing lxd containers on remote hosts via api

Lets assume we have two physical machines **localhost** and **remotehost** with ipv4 address 10.20.1.1, both having lxd installed, **localhost** having also ansible installed. Managing local and remote lxd containers from **localhost** differs in setting the ansible_host in the inventory file (/etc/ansible/hosts) according to this example:

```
[lxd]
localcontainer ansible_connection=lxd
remotecontainer ansible_connection=lxd ansible_host=remotehost:remotecontainer
```

In order for this to work, localhost's and remotehost's lxd have to be setup accordingly:

```
### Setup remotehost's lxd to listen on its IP and a nonstandard port 48443 to have additional security by obscurity
### (optionally use 0.0.0.0 to listen on all IPs). Define a trust_password and have a look at remotehost's certificate 
### fingerprint

remotehost $ sudo lxc config set core.https_address 10.20.1.1:48443 
remotehost $ sudo lxc config set core.trust_password MYSECRETPASS
remotehost $ sudo lxc info | grep certificate_fingerprint
  certificate_fingerprint: 6db21257d615902bbbd1c251335150dbe669ccc74523a0089b1017e15165aa41

### Generate a keypair and upload the client key to remotehost. Don't forget to check if the fingeprints
### match, otherwise you risk falling for an attacker's trap (i.e. a MITM attack).

localhost $ sudo lxc remote add remotehost 10.20.1.1:48443
Certificate fingerprint: 6db21257d615902bbbd1c251335150dbe669ccc74523a0089b1017e15165aa41
ok (y/n)? y
Admin password for remotehost: 
Client certificate stored at server:  remotehost

### Unset trust_password to prevent any possible brute-force attacks against lxd's api

remotehost $ sudo lxc config unset core.trust_password

### LXD's API is now secure good to go, it can be even exposed to the internet. The main security reason not to do so
### would be a lack of trust in the Go TLS stack or company policies. To summarize, to keep your LXD safe in production, 
### you'd most likely set core.https_address to a single address (not use 0.0.0.0), do yout usual firewall rules in
### front and in general not keep core.trust_password set after your clients are trusted. To review and manage LXD API's
### configuration, you might find the following helpful:

remotehost $ lxc config get core.https_address       # review listen ip and port
remotehost $ lxc config get core.trust_password      # review if trust_password is still set (returns true if set to something)
remotehost $ lxc config trust list                   # review all trusted clients
remotehost $ lxc config trust remove <fingerprint>   # revoke access to a client identified by <fingerprint>

localhost # lxc remote list
localhost # lxc remote list remotehost:
localhost # lxc list remotehost:
localhost # lxc exec remotehost:remotecontainer      # interact with a remotecontainer from localhost
```

Note that the "remotehost" in the inventory file is the name under which you see the machine with `lxc list remotehost:` - if you use a nonstandard port, i.e. `lxc remote add remotehost 10.20.1.1:48443`, then your inventory expects the given remotehost name, not the port or ip address.

Also see

* https://github.com/lxc/lxd/issues/3448
* https://github.com/lxc/lxd/issues/2098

