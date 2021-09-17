# Replicating Netbox DB from Production to Dev Server with Ansible

Sometimes there's the need to experiment before deploying to production. However, replicating the db everytime manually is boring and we would like to do it by running one command.

> **Side Note:** There are some packages needed for ansible to be able to work with the Postgres db - `acl` and `python-psycopg2`. I have included them in the playbook to be sure they are installed especially on the lab instance.

We have 3 machines we work with:

- Ansible controller
- Netbox Production server
- Netbox Dev server

The hosts file looks like this:

```text
[netbox_servers]
netbox_prod
netbox_dev

[netbox_servers:vars]
db_name=netbox
ansible_user=username
ansible_become_pass=sudo_password
ansible_python_interpreter=/usr/bin/python3
```

This mission can be accomplished in two ways:

## Fetch from Production and Push to Dev

This method involves copying to the ansible machine and then pushing to the dev server. This uses the fetch and copy modules.

You can use when conditional to limit which module will be run against which host.

```text
---
 - hosts: netbox
   tasks:
     - name: Fetch the file from the netbox_prod to master
       run_once: yes
       fetch:
         src: /backup/netbox_db
         dest: backups/netbox/
         flat: yes
       when: "{{ inventory_hostname == 'netbox_prod' }}"

     - name: Copy the file from master to netbox_dev
       copy:
         src: backups/netbox/netbox_db
         dest: /backups/
       when: "{{ inventory_hostname == 'netbox_dev' }}"
```

## Copy straight from Production to Dev Server

First, SSH Key-Based Authentication must be enabled between Production and Dev. There are 2 methods using the sync module - Sync Pull or Sync Push

### Sync Pull Method

This task is actually run on the destination host. The Dev server will pull from the Production Server. With the help of `delegate_to`, the synchronize module does not use the localhost by default and uses the destination host instead.

**Target**: netbox_prod  
**Delegation and Execution on**: netbox_dev  
**Mode**: Pull

```text
---
- name: Sync Pull task - Executed on  the Destination host "netbox_dev"
  hosts: "netbox_prod"
  tasks:
    - name: Copy the file from netbox_prod to netbox_dev using Method Pull
      tags: sync-pull
      synchronize:
        src: "{{ item }}"
        dest: "{{ item }}"
        mode: pull
      delegate_to: "netbox_dev"
      register: syncfile
      run_once: true
      with_items:
       - "/backup/netbox_db"
```

### Sync Push Method

This task is actually run on the sourcee host. The Prod server will push to the Dev Server. With the help of `delegate_to`, the synchronize module does not use the localhost by default and uses the netbox_prod host instead.

**Target**: netbox_dev  
**Delegation and Execution on**: netbox_prod  
**Mode**: Push

```text
---
- name: Sync Push task - Executed on  the Source host "netbox_prod"
  hosts: "netbox_dev"
  tasks:
    - name: Copy the file from netbox_prod to netbox_dev using Method Push
      tags: sync-push
      synchronize:
        src: "{{ item }}"
        dest: "{{ item }}"
        mode: push
      delegate_to: "netbox_prod"
      register: syncfile
      run_once: true
      with_items:
       - "/backup/netbox_db"
```

## References

<https://www.middlewareinventory.com/blog/how-to-copy-files-between-remote-servers-ansible-fetch-sync/>

<https://www.educba.com/ansible-synchronize/>
