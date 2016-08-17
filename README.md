# mage.lxd-provisioning

An ansible recipe to manage lxd containers on a host. Adding the following to a playbook called **test-lxd.yml** and running `ansible-playbook -b --ask-become-pass /home/killua/run-all.yaml` will:

- start a new container named 'mycontainer'
- add to the temporary in-memory inventory 'mycontainer' and set its ansible_connection to lxd
- test if python is present in the container, install it if not (doing this since python is a requirement to manage a node with ansible)
- add 'mycontainer' to your local /etc/ansible/hosts inventory, into the [lxd] group.

This recipe requires

- ansible 2.2 or newer
- lxd


**Designing lxd provisioning**

Setting up lxd containers is fairly straightforward thanks to the lxd plugin comming with ansible 2.2. The tough nut to crack is networking, because lxd doesn't have (at present) networking api in place. So what we do is the following:

- setup a nginx-based reverse proxy container (mageproxy) 
- upon lxd container creation,
  - get its assigned ip adress
  - use iptables (nat routing) to get the container connected
  - modify one of the following mageproxy files
    - /etc/hosts (add <ip> <containername>)
    - /etc/nginx/upstream.conf (do the same)


2. 

**test-lxd.yml playbook**

```
- hosts: localhost
  roles:
    - role: "mage.lxd-provisioning"
      lxd_provisioning_inventory:
        - name: "ubu_test_1"
          image: "ubuntu/xenial/amd64"
          nat:
            - extif: enp3s0
              intif: lxdbr0
              extport: 8080
              intport: 8080
          proxy:
            - name: "ubu1.vaizard.xyz"
              path: "/postgres/"
              pass: "
        - name: "ubu_test_2"
          image: "ubuntu/xenial/amd64"
          proxy:
            - "/sys/fs/cgroup:/sys/fs/cgroup:ro"
```

**using the lxd plugin without this role**
```
- hosts: localhost
  connection: local
  tasks:
    - name: Create a started container
      lxd_container:
        name: mycontainer
        state: started
        source:
          type: image
          mode: pull
          server: https://images.linuxcontainers.org
          protocol: lxd
          alias: ubuntu/xenial/amd64
        profiles: ["default"]
        wait_for_ipv4_addresses: true
        timeout: 600


    - name: add the host
      local_action: add_host name=mycontainer ansible_connection=lxd


    - name: check python is installed in container
      delegate_to: mycontainer
      raw: dpkg -s python
      register: python_install_check
      failed_when: python_install_check.rc not in [0, 1]
      changed_when: false

    - name: install python in container
      delegate_to: mycontainer
      raw: apt-get install -y python
      when: python_install_check.rc == 1

    - name: updating the inventory
      become: yes
      lineinfile: dest=/etc/ansible/hosts regexp="^mycontainer ansible_connection=lxd" insertafter="^\[lxd\]" line="mycontainer ansible_connection=lxd"
```
