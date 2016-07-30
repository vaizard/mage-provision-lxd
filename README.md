# mage.lxd-container

An ansible recipe to manage lxd containers on a host. Adding the following to a playbook called **test-lxd.yml** and running `ansible-playbook -b --ask-become-pass /home/killua/run-all.yaml` will:

- start a new container named 'mycontainer'
- add to the temporary in-memory inventory 'mycontainer' and set its ansible_connection to lxd
- test if python is present in the container, install it if not (doing this since python is a requirement to manage a node with ansible)
- add 'mycontainer' to your local /etc/ansible/hosts inventory, into the [lxd] group.

This recipe requires

- ansible 2.2 or newer
- lxd

**test-lxd.yml playbook**

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
