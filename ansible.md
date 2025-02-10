---
layout: page
title: Welcome
---

## Include files in directory as Var

Variable name will be name of file minus extension

```yaml
---
- hosts: localhost
  connection: local
  gather_facts: false
  become: false

  vars:
    validate_certs: true

  tasks:
    - name: Include fabricNode variables to get Switches
      ansible.builtin.include_vars:
        file: "{{ item }}"
        name: "{{ item | basename | splitext | first }}" # Set variable name to filename minus extension
      loop: "{{ query('ansible.builtin.fileglob','data/aci/*.json') }}" # Loop over all the files in directory
```
