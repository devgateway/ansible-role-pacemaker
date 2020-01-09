# Pacemaker role for Ansible

This role configures Pacemaker cluster by dumping the configuration (CIB), adjusting the XML, and
reloading it. The role is idempotent, and supports check mode.

It has been redesigned to configure individual elements (cluster defaults, resources, groups,
constraints, etc) rather than the whole state of the cluster and all the services. This allows you
to focus on specific resources, without interfering with the rest.

## Requirements

This role has been written for and tested in Scientific Linux 7. It might also work in other
distros, please share your experience.

## Tasks

Use `tasks_from` Ansible directive to specify what you want to configure.

Boolean values in properties (parsed by Pacemaker itself) don't have to be quoted. However,
resource agents may expect Boolean-like arguments as integers, strings, etc. Such values **must**
be quoted.

### `tasks_from: main`

Set up nodes, configure cluster properties, and resource defaults.

#### `pcmk_cluster_name`

Name of the cluster (optional).

Default: `hacluster`.

#### `pcmk_password`

The plaintext password for the cluster user (optional). If omitted, will be derived from
`ansible_machine_id` of the first host in the play batch. This password is only used in the initial
authentication of the nodes.

Default: `ansible_machine_id | to_uuid`

#### `pcmk_user`

The system user to authenticate PCS nodes with (optional). PCS will authenticate all nodes with
each other.

Default: `hacluster`

#### `pcmk_cluster_options`

Dictionary with [cluster-wide options](https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-cluster-options.html) (optional).

#### `pcmk_votequorum`

Dictionary with votequorum options (optional). See `votequorum(5)`. Boolean values accepted.

#### `pcmk_resource_defaults`

Dictionary of resource defaults (optional).

### `tasks_from: resource`

Configure a simple resource.

#### `pcmk_resource`

Dictionary describing a simple (*primitive*) resource. Contains the following members:

* `id`: resource identifier; mandatory for simple resources;
* `class`, `provider`, and `type`: resource agent descriptors; `provider` may be omitted, e.g. when
  `type` is `service`;
* `options`: optional dictionary of resource-specific attributes, e.g. address and netmask for
  *IPaddr2*;
* `op`: optional list of operations; each operation is a dictionary with required `name` and
  `interval` members, and optional arbitrary members;
* `meta`: optional dictionary of meta-attributes.

### `tasks_from: group`

Configure a resource group.

#### `pcmk_group`

Dictionary with two members:

* `id` is the group identifier
* `resources` is a dictionary where keys are resource IDs, and values have the same format as
  `pcmk_resource` (except for `id` of the resources being optional).

### `tasks_from: constraint`

Configure a constraint.

##### `pcmk_constraint`

Dictionary defining a single constraint. The following members are required:

* `type`: one of: `location`, `colocation`, or `order`;
* `score`: constraint score (signed integer, `INFINITY`, or `-INFINITY`).

Depending on the value of `type`, the following members are also required:

* `location` requires `rsc` and `node`;
* `colocation` requires `rsc` and `with-rsc`;
* `order` requires `first` and `then`;

The dictionary may contain other members, e.g. `symmetrical`.

## Example playbooks

### Active-active chrooted BIND DNS server

    ---
    - name: Configure DNS cluster
      hosts: dns-servers
      tasks:
    
        - name: Set up cluster
          include_role:
            name: devgateway.pacemaker
          vars:
            pcmk_password: hunter2
            pcmk_cluster_name: named
            pcmk_cluster_options:
              stonith-enabled: false
    
        - name: Configure IP address resource
          include_role:
            name: devgateway.pacemaker
            tasks_from: resource
          vars:
            pcmk_resource:
              id: dns-ip
              class: ocf
              provider: heartbeat
              type: IPaddr2
              options:
                ip: 10.0.0.1
                cidr_netmask: 8
              op:
                - name: monitor
                  interval: 5s
    
        - name: Configure cloned BIND resource
          include_role:
            name: devgateway.pacemaker
            tasks_from: advanced-resource
          vars:
            pcmk_resource:
              type: clone
              id: dns-clone
              resources:
                named:
                  class: service
                  type: named-chroot
                  op:
                    - name: monitor
                      interval: 5s
    
        - name: Set up constraints
          include_role:
            name: devgateway.pacemaker
            tasks_from: constraint
          vars:
            pcmk_constraint:
              type: order
              first: dns-ip
              then: dns-clone

### Active-active Squid proxy

    ---
    - name: Configure Squid cluster
      hosts: proxy-servers
      tasks:
    
        - name: Set up cluster
          include_role:
            name: devgateway.pacemaker
          vars:
            pcmk_password: hunter2
            pcmk_cluster_name: squid
            pcmk_cluster_options:
              stonith-enabled: false
    
        - name: Configure IP address resource
          include_role:
            name: devgateway.pacemaker
            tasks_from: resource
          vars:
            pcmk_resource:
              id: squid-ip
              class: ocf
              provider: heartbeat
              type: IPaddr2
              options:
                ip: 192.168.0.200
                cidr_netmask: 24
              op:
                - name: monitor
                  interval: 5s
    
        - name: Configure cloned BIND resource
          include_role:
            name: devgateway.pacemaker
            tasks_from: advanced-resource
          vars:
            pcmk_resource:
              id: squid
                type: clone
                resources:
                  squid-service:
                    class: service
                    type: squid
                    op:
                      - name: monitor
                        interval: 5s
    
        - name: Set up constraints
          include_role:
            name: devgateway.pacemaker
            tasks_from: constraint
          vars:
            pcmk_constraint:
              type: order
              first: squid-ip
              then: squid

### Nginx, web application, and master-slave Postgres

The cluster runs two Postgres nodes with synchronous replication. Wherever master is, a virtual IP
address is running, where NAT is pointing at. Nginx and the webapp are running at the same node, but
not the other, in order to save resources. Based on [the example from Clusterlabs
wiki](https://wiki.clusterlabs.org/wiki/PgSQL_Replicated_Cluster).

    ---
    - hosts:
        - alpha
        - bravo
      tasks:
    
        - name: Set up Pacemaker with Postgres master/slave
          include_role:
            name: devgateway.pacemaker
          vars:
            pcmk_pretty_xml: true
            pcmk_cluster_name: example
            pcmk_password: hunter2
            pcmk_cluster_options:
              no-quorum-policy: ignore
              stonith-enabled: false
            pcmk_resource_defaults:
              resource-stickiness: INFINITY
              migration-threshold: 1
    
        - name: Configure simple resources
          include_role:
            name: devgateway.pacemaker
            tasks_from: resource
          loop_control:
            loop_var: pcmk_resource
          loop:
            - id: coolapp
              class: service
              type: coolapp
            - id: nginx
              class: service
              type: nginx
            - id: virtual-ip
              class: ocf
              provider: heartbeat
              type: IPaddr2
              options:
                ip: 10.0.0.23
              meta:
                migration-threshold: 0
              op:
                - name: start
                  timeout: 60s
                  interval: 0s
                  on-fail: restart
                - name: monitor
                  timeout: 60s
                  interval: 10s
                  on-fail: restart
                - name: stop
                  timeout: 60s
                  interval: 0s
                  on-fail: restart
    
        - name: Configure master-slave Postgres
          include_role:
            name: devgateway.pacemaker
            tasks_from: advanced-resource
          vars:
            pcmk_resource:
              id: postgres
              type: master
              meta:
                master-max: 1
                master-node-max: 1
                clone-max: 2
                clone-node-max: 1
                notify: true
              resources:
                postgres-replica-set:
                  class: ocf
                  provider: heartbeat
                  type: pgsql
                  options:
                    pgctl: /usr/pgsql-9.4/bin/pg_ctl
                    psql: /usr/pgsql-9.4/bin/psql
                    pgdata: /var/lib/pgsql/9.4/data
                    rep_mode: sync
                    node_list: "{{ ansible_play_batch | join(' ') }}"
                    restore_command: cp /var/lib/pgsql/9.4/archive/%f %p
                    master_ip: 10.0.0.23
                    restart_on_promote: "true"
                    repuser: replication
                  op:
                    - name: start
                      timeout: 60s
                      interval: 0s
                      on-fail: restart
                    - name: monitor
                      timeout: 60s
                      interval: 4s
                      on-fail: restart
                    - name: monitor
                      timeout: 60s
                      interval: 3s
                      on-fail: restart
                      role: Master
                    - name: promote
                      timeout: 60s
                      interval: 0s
                      on-fail: restart
                    - name: demote
                      timeout: 60s
                      interval: 0s
                      on-fail: stop
                    - name: stop
                      timeout: 60s
                      interval: 0s
                      on-fail: block
                    - name: notify
                      timeout: 60s
                      interval: 0s
    
        - name: Set up constraints
          include_role:
            name: devgateway.pacemaker
            tasks_from: constraints
          vars:
            pcmk_constraints:
              - type: colocation
                rsc: virtual-ip
                with-rsc: postgres
                with-rsc-role: Master
                score: INFINITY
              - type: colocation
                rsc: nginx
                with-rsc: virtual-ip
                score: INFINITY
              - type: colocation
                rsc: coolapp
                with-rsc: virtual-ip
                score: INFINITY
              - type: order
                first: postgres
                first-action: promote
                then: virtual-ip
                then-action: start
                symmetrical: false
                score: INFINITY
              - type: order
                first: postgres
                first-action: demote
                then: virtual-ip
                then-action: stop
                symmetrical: false
                score: 0

## See also

- [The official Pacemaker documentation](http://clusterlabs.org/doc/)

## Copyright

Copyright 2015-2019, Development Gateway. Licensed under GPL v3+.
