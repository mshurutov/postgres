[TOC]

Role: postgres
==============

postgres is multifunctional role to deploy PostgreSQL in single, replica, HA-clusters (patroni,stolon) mode.

Copyright (C) 2023  Mikhail Shurutov

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.

Requirements
------------

This role requires python v3 because python v2 is out of live.

Role Variables
--------------

Role has many variables. For details see defaults/main.yml

Using Variables
------------

* `ANSIBLE_ROOT_DIR` is path for static content: roles,configs,etc, for example: /data/ansible
* `ANSIBLE_ROOT_ROLE_DIR` is path in `roles_path` config variable, for example: /data/ansible/roles
Content of my ~/.ansible.cfg:
```
...
# additional paths to search for roles in, colon separated
#roles_path    = /etc/ansible/roles
roles_path    = /data/ansible/roles
...
```

Install role
------------
### GIT repo

    user@host ~ $ cd $ANSIBLE_ROOT_ROLE_DIR
    user@host roles $ git clone https://git.code.sf.net/p/postgres-ansible-role/code postgres

### Ansible galaxy
#### Installation from command

    user@host ~ $ cd $ANSIBLE_ROOT_DIR
    user@host ansible $ ansible-galaxy role install mshurutov.postgres -p roles

#### Installation from requirements.yml

    user@host ~ $ cd $ANSIBLE_ROOT_DIR
    user@host ansible $ grep postgres requirements.yml
    - name: mshurutov.postgres
    user@host ansible $ ansible-galaxy role install -r requirements.yml -p roles

Deploy postgres using this role
------------

On every case you must set OS-depended variables:

* `postgres_home_dir` is home directory of system user owner of instance;
* `postgres_conf_dir` is directory where config files are placed;
* `postgres_data_dir` is $PGDATA directory;
* `postgres_bin_dir` is directory where executable files is placed by package manager;
* `postgres_service_name` is systemd service name of postgres.

Default values are set for Gentoo Linux.

### Setup single instance
#### Set role variables

You may need set `postgres_major_version` only in this mode. Default value of this variable is `14`.

#### Playbook example

Role is installed by ansible galaxy (see above) in this example.

    ...
    - hosts: pg_host
      roles:
         - role: mshurutov.postgres
           tags: postgres
    ...

#### Deploy postgres

    ansible-playbook postgres-playbook.yml -t postgres -l pg_host -D

### Setup replica
#### Set role variables

For use this mode you must define follow variables: `postgres_install_mode` should be set to "`replica`"; `postgres_master_host`, `postgres_backup_connstring`, `postgres_replica_init` need to be set to `"from_master"` `postgres_recovery_params` as shown below (`{{ common_domain_name }}: 'example.com'` is set on `group_vars/all`):

```
user@ansible_host pgdev $ cat host_vars/pg_replica.yml
# Common settings
common_short_hostname: "pg_replica"
common_full_hostname: "{{ common_short_hostname }}.{{ common_domain_name }}"
common_ip4_default: "192.168.1.12"
# postgres settings
postgres_install_mode: "replica"
postgres_master_host: "pg_master.{{ common_domain_name }}"
postgres_backup_connstring: "-h {{ postgres_master_host }} -U {{ postgres_repl_user }}"
postgres_replica_init: "from_master"
postgres_recovery_params:
  - { name: "primary_conninfo", value: "'host={{ postgres_master_host }} user={{ postgres_repl_user }}'" }
  - { name: "hot_standby", value: "on" }
user@ansible_host pgdev $
```

#### Playbook example

Role is installed by git (see above) in this example.

    ...
    - hosts: pg_replica
      roles:
         - role: postgres
           tags: postgres
    ...

#### Deploy postgres

    ansible-playbook postgres-playbook.yml -t postgres -l pg_replica -D

### Configure postgres HA-clusters using patroni

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
postgres_install_mode: "patroni"
...
postgres_master_host: "srv01dev.{{ common_domain_name }}"
# Patroni settings
patroni_config_file: '{{ patroni_config_dir }}/config.yml'
patroni_scope_name: 'clusterdev'
patroni_log_params:
  - name: "level"
    value: "DEBUG"
  - name: "dir"
    value: "/var/log/postgresql"
```
`patroni_config_file` is distro depened variable;

#### Configure patroni cluster with etcd as DCS

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
etcd_cluster_group: "dev"
...
postgres_cluster_dcs: "etcd"
...
```
#### Configure patroni cluster with consul as DCS

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
# Consul settings
postgres_consul_hosts_group: "dev"
...
postgres_cluster_dcs: "consul"
...
```

### Configure postgres HA-clusters using stolon

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
postgres_install_mode: "stolon"
...
# Stolon settings
stolon_apt_repo: "deb [trusted=yes] http://srv01/repos/deb/ bullseye main"
stolon_daemons_common_options_group:
  - key: "log_level"
    value: "info"
    comment: "# debug, info (default by developer), warn or error (default by me)"
  - key: "store_backend"
    value: "{{ stolon_store_backend }}"
    comment: |
      # store backend type (etcdv2/etcd, etcdv3, consul or kubernetes)
stolon_daemons_proxy_options_group:
  - key: "port"
    value: "7432"
    comment: "# proxy listening port (default \"5432\")"

...
```
#### Configure stolon cluster with etcd as DCS

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
stolon_store_backend: "etcd"
...
```

#### Configure stolon cluster with consul as DCS

```
mshurutov@hpbook02 pgdev $ cat group_vars/dev.yml
---

# Variables for group dev
...
stolon_store_backend: "consul"
...
```

### Example of playbook for deploy PostgreSQL HA cluster

    ...
    - hosts: pg_dcs_hosts
      roles:
         - role: mshurutov.etcd
           tags: pg_cluster,pg_dcs,etcd
    ...
    - hosts: pg_hosts
      roles:
         - role: mshurutov.postgres
           tags: pg_cluster,postgres
    ...

### Deploy postgres HA cluster

    ansible-playbook postgres-playbook.yml -t pg_cluster -D

License
-------

[GPLv2](https://www.gnu.org/licenses/old-licenses/gpl-2.0.txt)


Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
