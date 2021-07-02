# Selfhost Forem

This is a repo for setting up a single install of [Forem](https://github.com/forem/forem). It uses [Fedora CoreOS](https://getfedora.org/en/coreos) as the host operating system and containers powered by [Podman](https://podman.io/) with [systemd](https://systemd.io/). We use [Ansible](https://github.com/ansible/ansible) to define your Forem in code and [Butane](https://coreos.github.io/butane/) / [Ignition](https://coreos.github.io/ignition/) to provision the system on boot. The Ignition configuration provided by this repo is a derivative of what we use for our Forem Cloud service.

We currently support three cloud providers: [DigitalOcean](https://www.digitalocean.com/), [AWS](https://aws.amazon.com/), and [Google Cloud](https://cloud.google.com/). We also support booting a development VM locally on Linux via [QEMU](https://www.qemu.org/). You can also use the repo to create a Butane YAML file to customize to fit your needs or bootable Ignition configuration to consume on bare metal or in a custom VM.

For those that want to DIY, you can use the systemd units in the [Butane template](https://github.com/forem/selfhost-devel/blob/main/playbooks/templates/forem.yml.j2) as an example of how to run Forem without Fedora CoreOS on a Linux distribution that supports systemd.

The goal of this project is to provide you with the choice, freedom, and cost-effectiveness to host your own Forem community as you see fit. 

We can't wait to see the community you selfhost with Forem!

## Requirements

- Git
- [Python 3.x](https://www.python.org/downloads/) and pip3
    - Mac OS: `brew install pip3`
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html): `ansible-core` 2.11 or greater (provided by Ansible 4.0.0)
- [Butane](https://github.com/coreos/butane/blob/master/docs/getting-started.md#getting-butane)
    - Mac OS: `brew install butane`
- pwgen
    - Mac OS: `brew install pwgen`
- Fedora CoreOS [34.20210427.2.1](https://getfedora.org/en/coreos?stream=stable) or greater
- A supported cloud provider, bare metal server, or a VM in QEMU.

*Note: Some provisioning targets have additional requirements that are detailed out in each respective section.*

## Quick Start

_Note: Following this quick start guide with the cloud provider of your choice will cost you money! Please consult with each cloud provider to figure out how much your Forem will cost you per month._

1) Clone the forem/selfhost repo to your local computer:
     - `git clone https://github.com/forem/selfhost.git`
2) Change into the selfhost directory: `cd selfhost`
3) Install Requirements:
    - `pip3 install -r requirements.txt`
4) Generate an Ansible Vault password
    - `pwgen -1 24|tee ~/.forem_selfhost_ansible_vault_password`
5) Copy example Ansible Inventory from `inventory/example/setup.yml` to `inventory/forem/setup.yml`
6) Edit `inventory/forem/setup.yml` Ansible Inventory with your Forem settings
    - Edit the following Ansible inventory variables:
        - default_email (Admin Email for system to use)
        - forem_domain_name (A domain name that you own and set A records on at your DNS provider)
        - forem_subdomain_name (defaults to www)
        - forem_server_hostname (defaults to host)
    - Generate and save Ansible Inventory secrets using [ansible-vault encrypt_string](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-individual-variables-with-ansible-vault) (see Required Ansible Vault secret variables in setup.yml):
        - vault_secret_key_base
        - vault_imgproxy_key
        - vault_imgproxy_salt
        - vault_forem_postgres_password
7. Generate a [SSH key](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key) and save it to `${HOME}/.ssh/forem`. Use `ls -lh ~/.ssh/forem*` to ensure you have both a `${HOME}/.ssh/forem` private key, and a corresponding `${HOME}/.ssh/forem.pub` public key.
8) Pick a supported cloud provider and set it up on your workstation
    - [AWS](https://github.com/forem/selfhost#aws)
    - [DigitalOcean](https://github.com/forem/selfhost#digitalocean)
    - [Google Cloud](https://github.com/forem/selfhost#google-compute)
9) Run the Ansible Playbook for your chosen cloud provider
    - [AWS](https://github.com/forem/selfhost#provision)
    - [DigitalOcean](https://github.com/forem/selfhost#provision-1)
    - [Google Cloud](https://github.com/forem/selfhost#provision-2)
10) Once your Forem VM is set up with your chosen cloud provider, you will need to point DNS at the IP address that is output at the end of the provider playbook.
11) Once DNS is pointed at your Forem VM, you will need to restart the Forem Traefik service (`sudo systemctl restart forem-traefik.service`) [via SSH on your Forem server](https://github.com/forem/selfhost#ssh-examples) to generate a TLS cert.
12) Go to your Forem domain name and create your first account. Please see the Forem Admin documentation located [here](https://forem-admin.netlify.app/) for more information on setting up your Forem.

----

## Provisioning Targets

**Note about recommended instance types and cost:** for each hosted provisioning target below, we attempted to recommend an instance type with 2 CPUs, 2GB of RAM, and a monthly cost of around $15 USD. Please note that providers may charge additionally for disk space, network usage, etc, so your price per month may vary based on your Forem's usage and needs. For exact and specific pricing information, please see each provider directly.

----
### AWS

The AWS provisioning target has a few preset variables that can be either edited in the playbook or passed along as Ansible [extra vars](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#defining-variables-at-runtime) on the CLI.  

```
fcos_aws_region: us-east-1
fcos_aws_size: t3a.small
fcos_aws_ebs_size: 100
ssh_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

- `fcos_aws_region`: the AWS region that is used to setup your Forem server. The default region is in `us-east-1` which is in North Virginia, USA
- `fcos_aws_size`: the AWS EC2 instance type. A recommended type is a `t3a.small` EC2 instance, with 2 VCPUs and 2GB of RAM
- `fcos_aws_ebs_size`: the amount of EBS disk space (in GB)
- `ssh_key`: the path to a public SSH key. Note that AWS's EC2 service can only use RSA based SSH keys. If you get an error that your SSH key is not the right type, please generate an RSA based SSH key and set `ssh_key` with a lookup path to that key

#### Setup
1) Install the [Ansible Amazon AWS collections](https://github.com/ansible-collections/amazon.aws) `ansible-galaxy collection install amazon.aws community.aws` or install them via `ansible-galaxy collection install -r requirements.yml`
2) Download and install the [AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) tool
3) Install `boto`, `boto3`, and `botocore` pip modules `pip3 install boto boto3 botocore` or run `pip install -r requirements.txt`
4) [Create an AWS IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with Programmatic access called `forem-selfhost` with the following `AmazonEC2FullAccess`, `AmazonS3FullAccess`, `AmazonVPCFullAccess` AWS managed policies attached. Be sure to save the Access key ID and Secret access key to use in step 5.
5) Run `aws configure --profile forem-selfhost` and input the access key ID and secret key when prompted. We use `us-east-1` for default region name but you can choose a different one if you wish. Set default output format to `json`

#### Provision
1) Run the AWS provider playbook to setup your Forem
    - `ansible-playbook -i inventory/forem/setup.yml playbooks/providers/aws.yml`

----
### DigitalOcean

The DigitalOcean provisioning target has a few preset variables that can be either edited in the playbook or pass along as Ansible [extra vars](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#defining-variables-at-runtime) on the CLI.

```
forem_do_region: nyc3
forem_do_size: s-2vcpu-2gb
```

- `forem_do_region`: the DigitalOcean region that is used to setup your Forem server. The default region is `nyc3` which is in New York City, New York, USA
- `forem_do_size`: the Droplet size. The recommended size is `s-2vcpu-2gb`, with 2 Shared CPUs, 2GB of RAM, a 60GB SSD disk, and 3TB of outbound transfer.

#### Setup
1) Install the [DigitalOcean Ansible collection](https://github.com/ansible-collections/community.digitalocean) `ansible-galaxy collection install community.digitalocean` or install it via `ansible-galaxy collection install -r requirements.yml`
2) [Download and install](https://docs.digitalocean.com/reference/doctl/how-to/install/) `doctl`
3) [Create DigitalOcean Auth Token](https://docs.digitalocean.com/reference/api/create-personal-access-token/)
4) Run `doctl auth init --access-token APITOKEN` and pass the API token created from step 3 and verify that you can authenticate to the DigitalOcean API with `doctl account get`

#### Provision
1) Run the DigitalOcean provider playbook to set up your Forem
    - `ansible-playbook -i inventory/forem/setup.yml playbooks/providers/digitalocean.yml`

**Note**: DigitalOcean does not have support for Fedora CoreOS. We have to upload a custom image to your account via Ansible. If the "`Wait for fcos-{{ fcos_download_release }} to be created`" task times out. please check the [Custom Images](https://cloud.digitalocean.com/images/custom_images) section on your DigitalOcean account to see if your image is still in a pending state. Wait for it to finish processing and re-run the DigitalOcean provider playbook.

----
### Google Cloud

The Google Cloud provisioning target has a few preset variables that can be either edited in the playbook or pass along as Ansible [extra vars](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#defining-variables-at-runtime) on the CLI.

```
forem_gcp_region: us-central1
forem_gcp_zone: a
forem_gcp_machine_type: e2-small
forem_gcp_disk_size: 100
forem_gcp_project: forem-selfhost
```

- `forem_gcp_region` + `forem_gcp_zone`: the Google Cloud region and zone that is used to setup your Forem server. The default region is `us-central1` in zone `a` which is in Council Bluffs, Iowa, USA
- `forem_gcp_machine_type`: the GCP machine type. A recommended type is `e2-small`, with 2 shared CPUs and 2GB of RAM
- `forem_gcp_disk_size`: the amount of disk space (in GB)
- `forem_gcp_project`: your GCP project name

#### Setup
1) Install the Google Cloud collection `ansible-galaxy collection install google.cloud` or install it via `ansible-galaxy collection install -r requirements.yml`
2) Install `requests` and `google-auth` pip modules `pip install requests google-auth` or run `pip3 install -r requirements.txt`
3) Create a [Google Cloud Service Account](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#creatinganaccount) called `forem-selfhost` with Compute Instance Admin (v1) privileges and [download a JSON credentials file](https://support.google.com/cloud/answer/6158849?hl=en&ref_topic=6262490#serviceaccounts) and place it in `~/.gcp/forem.json`

#### Provision
1) Run the Google Cloud provider playbook to setup your Forem
    - `ansible-playbook -i inventory/forem/setup.yml playbooks/providers/gcp.yml`

----
## Ansible Dynamic Inventories

We provide some example [Dynamic Inventories](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html) for you to use on your selfhosted Forem. You can use them to run [Ansible Adhoc Commands](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html) or write your own Ansible playbooks to manage your Forem.

See the [SSH Examples](https://github.com/forem/selfhost#ssh-examples) for some commands that you can run with an Ansible Adhoc command.

### AWS

*Show all Forems on AWS*
```
ansible-inventory -i inventory/providers/aws/ --graph forem

```

*Run an Ansible Adhoc command on all Forems on AWS*
```
ansible -i inventory/providers/aws/ -m command -a "hostname" forem

```
### DigitalOcean

_Note: You need to run `export DO_API_TOKEN=your_digitalocean_api_token` before running the `ansible-inventory` or `ansible` commands!_

*Show all Forems on DigitalOcean*
```
ansible-inventory -i inventory/providers/digitalocean/ --graph forem

```

*Run an Ansible Adhoc command on all Forems on DigitalOcean*
```
ansible -i inventory/providers/digitalocean/ -m command -a "hostname" forem

```

### Google Compute

_Note: You need to edit the `project` list in `inventory/providers/gcp/gcp.yml` with your GCP project for this Ansible Inventory Dynamic to work correctly!_

*Show all Forems on Google Compute*
```
ansible-inventory -i inventory/providers/gcp/ --graph forem

```

*Run an Ansible Adhoc command on all Forems on Google Compute*
```
ansible -i inventory/providers/gcp/ -m command -a "hostname" forem

```

----
## Configuration Internals

This section covers how Forem is configured and run on Fedora CoreOS.

### systemd
Forem is run with a stack of containers that are powered via [Podman](https://podman.io/) and [systemd](https://systemd.io/). The systemd unit files are located in `/etc/systemd/system`:

```
$ cd /etc/systemd/system
$ ls -lah forem*
-rw-r--r--. 1 root root  243 Jun 29 17:16 forem-container.service
-rw-r--r--. 1 root root  833 Jun 29 17:16 forem-imgproxy.service
-rw-r--r--. 1 root root 1.1K Jun 29 17:16 forem-openresty.service
-rw-r--r--. 1 root root  787 Jun 29 17:16 forem-pod.service
-rw-r--r--. 1 root root  904 Jun 29 17:16 forem-postgresql.service
-rw-r--r--. 1 root root 1.4K Jun 29 17:16 forem-rails.service
-rw-r--r--. 1 root root  941 Jun 29 17:16 forem-redis.service
-rw-r--r--. 1 root root  951 Jun 29 17:16 forem-traefik.service
-rw-r--r--. 1 root root 1006 Jun 29 17:16 forem-worker.service
-rw-r--r--. 1 root root  691 Jun 29 17:16 forem.service
```

We use [systemd unit dependencies](https://fedoramagazine.org/systemd-unit-dependencies-and-order/) heavily to correctly configure the start of service required to power your Forem.

The first systemd unit that runs on boot is `forem-container.service`. This service interfaces with `foremimg` to pull down the Forem container image and ensure that the `localhost/forem/forem:current` container tag is present.

We then create a [Podman pod](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods) with the `forem-pod.service` to run all of the Forem services within it. Pods are a group of one or more containers that share the same network, pid and ipc namespaces. This means that `localhost` is isolated inside the pod and shared across all of the containers within the pod. The pod also binds ports `80` and `443` on the Fedora CoreOS server.

We then launch all of the required services within this Podman pod: `forem-imgproxy.service`, `forem-postgresql.service`, `forem-redis.service`. All of these services use the systemd [BindsTo](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#BindsTo=) directive which configures a strong dependency on `forem-pod.service`. This means if `forem-pod.service` is stopped or it enters an inactive state, all of these services will stop too. Also, all of these services have to be up before our next unit, `forem.service` can start successfully.

The `forem.service` unit uses the [BindsTo](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#BindsTo=) directive to bind `forem-rails.service`, `forem-worker.service`, and `forem-openresty` together as they are tightly dependent on each other to run Forem. This means you can stop and start the `forem.service` unit via `systemctl` and it will stop the three main Forem units, too. This service also ensures that `forem-pod.service`, `forem-postgresql.service`, `forem-imgproxy.serivce`, and `forem-redis.service` units are active and that `image exists localhost/forem/forem:current` exists before starting.

The main Forem systemd units are `forem-rails.service`, `forem-worker.service`, and `forem-openresty`. The `forem-rails.service` creates container volume mount `/opt/forem/data/uploads` on the Fedora CoreOS host and mounts it inside the container at `/opt/apps/forem/public/uploads`. The Forem [Containerfile](https://github.com/forem/forem/blob/main/Containerfile#L76) uses a `VOLUME` directive to create a container volume `/opt/apps/forem/public` and it puts all of the Forem public assets (CSS and JS) inside. These volumes are shared between the other main Forem containers: `forem-worker.service`, and `forem-openresty`.

The `forem-rails.service` is the main Ruby on Rails application that is running [Puma](https://puma.io/) which is a very fast and concurrent HTTP 1.1 application server.

The `forem-worker.service` is the background worker container that runs [Sidekiq](https://github.com/mperham/sidekiq).

The `forem-openresty.service` runs OpenResty which is a dynamic web platform based on NGINX and LuaJIT. We use OpenResty to proxy connections to Puma in the `forem-rails.service` unit. We also use OpenResty to send proxy requests to `forem-imgproxy.service` for image resizing, which are then cached in OpenResty.

The last service we use in the configuration phase is `forem-traefik.service`. It is responsible for handling traffic from the Internet and passing it into the Forem Pod to `forem-openresty.service`, which then manages the traffic to `forem-rails.service`. It also manages the TLS certificate from [Let's Encrypt](https://letsencrypt.org/docs/) and handles the redirection from HTTP to HTTPS.

### Forem configs

All of your Forem's data and configuration resides in `/opt/forem`. This is the most important directory on your Forem. You should backup this directory regularly.

```
# ll /opt/forem/
total 4
drwxr-x---. 3 root root 39 Jun 29 17:16 configs
drwxr-x---. 5 root root 52 Jun 29 17:16 data
drwxr-x---. 2 root root 82 Jun 29 17:16 envs
drwxr-xr-x. 2 root root  6 Jun 29 17:16 tmp
-rw-r--r--. 1 root root 37 Jun 29 17:16 version

```

The `configs` directory contains the OpenResty (Nginx) configuration file and the Traefik configuration TOML files, along with the `acme.json`, which holds the TLS certificate from Let's Encrypt.

```
# ll
total 4
-rw-r--r--. 1 root root 2375 Jun 29 17:16 nginx.conf
drwxr-x---. 2 root root   63 Jun 29 17:16 traefik
# ll traefik/
total 12
-rw-------. 1 root root 3524 Jun 29 17:20 acme.json
-rw-r--r--. 1 root root 1928 Jun 29 17:16 dynamic.toml
-rw-r--r--. 1 root root  740 Jun 29 17:16 traefik.toml
```

The `data` directory contains your `postgresql`, `redis`, and `upload` directories. This directory contains all of your Forem content including your members.

```
# ll data/
total 4
drwx------. 19 polkitd root 4096 Jun 29 17:19 postgresql
drwxr-x---.  2 polkitd root   28 Jun 29 17:19 redis
drwxr-x---.  2 core    core    6 Jun 29 17:16 uploads

```

The `envs` directory contains all of the environment variable files that configure the following Forem services:

* `forem-imgproxy.service` with `imgproxy.env`
* `forem-postgresql.service` with `postgresql.env`
* `forem-rails.service`, `forem-worker.service` with `rails.env`
* `forem-redis.service` with `redis.env`

```
# ll envs/
total 12
-rw-r-----. 1 root root  319 Jun 29 17:16 imgproxy.env
-rw-r-----. 1 root root   85 Jun 29 17:16 postgresql.env
-rw-r-----. 1 root root 1483 Jun 29 17:16 rails.env
-rw-r-----. 1 root root    0 Jun 29 17:16 redis.env

```

If you have to make a configuration to a service, you can edit the respective ENV file and restart the service via systemd. For example, `systemctl restart forem.service` after editing the `rails.env` file.

The `version` file is written out by the `forem-container.service` systemd unit and `foremimg` script.

_Note: Making changes to these files can prevent your Forem from starting and cause downtime. Make changes with care and create backups!_


## SSH Examples

All of these examples need to be run via SSH on the Fedora CoreOS server as the `core` or `root` user. You can access your Forem server via SSH:

```
ssh core@<SERVER IP ADDRESS>
```

### foremctl

We have a helper script (Forem Control) called `foremctl`. It is used to control your Forem via CLI.

```
$ foremctl help

Usage: foremctl {console|deploy|help|rake|restart|start|stat|status|stop|update|version}

console         Open a Rails console
deploy          Updates and deploy the most current version of Forem
help            Show this message
rake            Run a rake task
restart         Restart Forem
start           Start Forem
stats           Show CPU, RAM, Disk IO usage of the Forem containers
status          Show the current running Forem containers
stop            Stop Forem
update          Updates Forem to the lastest container
version         Shows information on the current running version of Forem
```

### Update Forem to the latest version and restart

```
sudo foremctl deploy
```

_Note: The deploy process causes a small amount of downtime while the Forem code restarts._


### foremimg

We have a helper script (Forem Image) called `foremimg`. It is used to control your Forem's version and apply updates.  It has to be run as the `root` user.

```
# foremimg help

Usage: foremimg {help|rollback|show}

help            Show this message
rollback        Issue a rollback

show            Show tags: current|rollback
  current       Show what image is tagged with current
  rollback      Show what image is tagged with rollback

Running foremimg with no flags will read /opt/forem/version if present for the
container tag or write /opt/forem/version with the default tag quay.io/forem/forem:latest

Running foremimg quay.io/forem/forem:testing will write quay.io/forem/forem:testing
to /opt/forem/version and pull this container and point the local container
image tag 'localhost/forem/forem:current' to 'quay.io/forem/forem:testing'
and point the previous image to 'localhost/forem/forem:rollback'

Rollbacks:
Running 'foremimg rollback' will swap 'localhost/forem/forem:rollback' with 'localhost/forem/forem:current'

```

### Set the Forem container repository and tag

```
sudo foremimg quay.io/forem/forem:testing

```


### Update Forem to the latest version with no restart

```
sudo foremimg update

```

### Rollback Forem to the last running version and restart

```
sudo foremimg rollback
sudo foremctl restart

```

### Update Fedora CoreOS to the latest stable version

Check for updates:

```
$ sudo rpm-ostree upgrade --check
2 metadata, 0 content objects fetched; 16 KiB transferred in 1 seconds; 0 bytes content written
AvailableUpdate:
        Version: 34.20210529.1.0 (2021-06-01T19:22:39Z)
         Commit: 936a0a142a09ebf8fa25d50a93377d8822c4ab3bfcf477a73781823569dbd33f
   GPGSignature: Valid signature by 8C5BA6990BDB26E19F2A1A801161AE6945719A39
           Diff: 380 upgraded, 22 removed, 17 added
```

Preview the package updates:

```
$ sudo rpm-ostree upgrade --preview
```

Download the packages without deploying them:

```
$ sudo rpm-ostree upgrade --download-only
```

To apply the updates and reboot:

```
$ sudo rpm-ostree upgrade
$ systemctl reboot
```
_Note: Fedora CoreOS is an immutable Linux distribution. You will have to reboot to have the updates take effect. This will cause downtime for your Forem._

If the update causes issues with your Forem, you can issue a rollback with:

```
sudo rpm-ostree rollback --reboot
```

### Backup your Forem data

You can make a backup of your Forem data by creating gzipped tarball of `/opt/forem`. You will want to download this file to your local computer via SCP.

```
foremctl stop
sudo tar czf ~core/"$(date '+%Y-%m-%d')-forem-data.tar.gz" /opt/forem
foremctl start
```
_Note: Running `foremctl stop` will cause downtime for your Forem!_

----
## Development

To support local development, you will need a Linux workstation with virtualization
enabled in the BIOS and KVM installed. You can check your workstation to see if it has
support for Intel VT/AMD-V Virtualization with this command:

```bash
grep --color "svm\|vmx" /proc/cpuinfo
```

If that doesn't return anything, you might need to enable virtualization support
in your BIOS.

### Install Dependencies

In order to set up your development VM, you'll need a Linux workstation or server
with the following packages:

- Git
- [Python 3.x](https://www.python.org/downloads/) and pip3
- Ansible 2.10 or greater (`pip3 install -r requirements.txt`)
- [Butane](https://github.com/coreos/butane/blob/master/docs/getting-started.md#getting-butane)
- [libvirt](https://libvirt.org/docs.html)
- [QEMU](https://qemu-project.gitlab.io/qemu/)
- `virt-install`
- [podman](https://podman.io/getting-started/installation)

**Fedora**

```bash
$ sudo dnf install -y @virtualization butane podman
```

**RHEL/CentOS**

```bash
$ sudo yum install -y qemu-kvm virt-install virt-manager podman
```

**Ubuntu**

```bash
$ sudo apt install virt-manager
```

**Debian**

```bash
$ sudo apt install qemu-kvm libvirt-bin
```

#### SSH Key

Before creating your development VM, you must supply an SSH key. SSH is the
primary method you'll use to interact with the development VM. Ansible will look
for a public key at the following path `/${HOME}/.ssh/forem.pub`.
Creating a symbolic link to your SSH key is a good way to handle this:

```bash
ln -s <path_to_your_private_ssh_key> ${HOME}/.ssh/forem.pub
```

#### SSH Config

Add this to your `~/.ssh/config` file:

```
Host devel.forem.wtf
  ForwardAgent yes
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
```

#### Ansible Vault

In order to encrypt your variables, you need an Ansible Vault password. This
password needs to be set once and never changed. If you lose this password or
change this password you will need to reset all of your encrypted variables.

```bash
pwgen -1 35 | tee ~/.forem_selfhost_ansible_vault_password
```

#### Create Forem Ansible Inventory

1) Copy example Ansible Inventory from `inventory/example/setup.yml` to `inventory/forem/setup.yml`
2) Edit `inventory/forem/setup.yml` Ansible Inventory with your Forem settings
    - Edit the following Ansible Inventory variables:
        - default_email
        - forem_domain_name
        - forem_subdomain_name
        - forem_server_hostname
    - Generate and save Ansible Inventory secrets:
        - vault_secret_key_base
        - vault_imgproxy_key
        - vault_imgproxy_salt
        - vault_forem_postgres_password

#### Launch Forem Locally

```bash
ansible-playbook -i inventory/forem/setup.yml playbooks/providers/qemu.yml
```

#### Local SSH Access

```bash
ssh core@devel.forem.wtf -p 2222
```
