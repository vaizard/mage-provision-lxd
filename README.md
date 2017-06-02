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
      provisioning_inventory:
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
