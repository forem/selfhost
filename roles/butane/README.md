Butane
=========

Transpile a Butane YAML file (.bu) into an Ignition JSON (.ign) file

Requirements
------------

This requires the [butane](https://github.com/coreos/butane) binary to be installed locally.

Role Variables
--------------

- butane_input_template: fcc.yml.j2
- butane_compress: false
- butane_b64encode: false


Dependencies
------------

Example Playbook
----------------

    tasks:
      - name: Generate FCC
        include_role:
          name: butane
          tasks_from: butane
        vars:
          fcc_file: "my_cool_butane_template.yml.j2"

License
-------

GNU Affero General Public License v3.0

Author Information
------------------

Forem Systems Engineering
