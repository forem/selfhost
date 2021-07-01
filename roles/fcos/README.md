FCOS
====

This role will gather facts about Fedora CoreOS Streams and download FCOS images

Requirements
------------

Unxz and gzip are required to use the extract tasks.

Role Variables
--------------

- fcos_stream: stable
- fcos_arch: x86_64
- fcos_platform: qemu
- fcos_format: "qcow2.xz"
- fcos_storage_location: "{{ playbook_dir }}/images"
- fcos_extraction_location: "$HOME/.local/share/libvirt/images"
- fcos_build_url: "https://builds.coreos.fedoraproject.org/streams/{{ fcos_stream }}.json"
- fcos_gpg_key_location: "https://getfedora.org/static/fedora.gpg"
- fcos_aws_region: us-east-2

Dependencies
------------

None

Example Playbook
----------------

Here are some examples on how to include tasks from this role in your playbooks:

**Gather and set some FCOS facts**

    - name: Get FCOS facts
      include_role:
        name: fcos
        tasks_from: facts


License
-------

GNU Affero General Public License v3.0

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
