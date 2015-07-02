# Pacemaker role for Ansible

## Properties

Ansible variables with underscores correspond to pacemaker properties with hyphens.

**Be sure to quote cluster properties!** By default, YAML parser will guess variable types, so the string "false" will be converted to Boolean False and then to string "False". Pacemaker properties are case-sensitive, e.g. "stonith-enabled=False" will be accepted, but STONITH will still be on.

Correct example:

    pacemaker_properties:
      stonith_enabled: "false"
