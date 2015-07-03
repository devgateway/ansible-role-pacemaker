# Pacemaker role for Ansible

## Requirements

This role has been written for and tested on Scientific Linux 7, so it should be applicable to Fedora 19. It might also work in other distros, please share your experience.

## Role variables

### pacemaker\_ansible\_group

The Ansible inventory group containing cluster peers. Since a host be a member of multiple and nested groups, we can't reliable guess this value.

The playbook will walk the group members and configure them as members of a cluster.

### pacemaker\_cluster\_name

Name of the cluster.

### pacemaker\_package

Package containing PCS daemon and client, which also depends on Pacemaker and Corosync packages. In EL and Fedora the package is called *pcs*.

### pacemaker\_user

The system user to authenticate PCS nodes with. PCS will authenticate all nodes with each other.

### pacemaker\_password

The plaintext password for the mentioned user. It will be hashed with per-host salt to maintain idempotency.

### pacemaker\_properties

The keys of this dict/hash with underscores correspond to pacemaker properties with hyphens.

**Be sure to quote cluster properties!** By default, YAML parser will guess variable types, so the string "false" will be converted to Boolean False and then to string "False". Pacemaker properties are case-sensitive, e.g. "stonith-enabled=False" will be accepted, but STONITH will still be on.

Correct example:

    pacemaker_properties:
      stonith_enabled: "false"

### pacemaker\_resources

An array of resource definitions. Each definition is a dict of two mandatory members, *id* (resource name) and *type* (standard:provider:type string, see output of *pcs resource providers*).

They can also have optional members like *options* dict, *op* list with operation actions and their options.

Additionally, there might be mutually exclusive members: Boolean *clone*, or dicts *masterslave* or *group* with their respective options. 

Finally, the values *disabled* and *wait* might be present.

For the detailed descriptions check out the resources below.

## Examples

### Inventory

    [cluster-dns]
    alpha
    bravo

### Playbook
    ---
    - hosts: cluster-dns
      roles:
        - pacemaker
      vars:
        pacemaker_ansible_group: cluster-dns
        pacemaker_password: secret
        pacemaker_cluster_name: foobar
        pacemaker_properties:
          stonith_enabled: "false"
        pacemaker_resources:
          - id: dns-ip
            type: "ocf:heartbeat:IPaddr2"
            options:
              ip: 10.0.0.1
              cidr_netmask: 8
            op:
              - action: monitor
                options:
                  interval: 5s
          - id: dns-srv
            type: "systemd:named-chroot"
            op:
              - action: monitor
                options:
                  interval: 5s
            clone: true

## See also

- [The official Pacemaker documentation](http://clusterlabs.org/doc/)
- *man pcs(8)*
