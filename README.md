# Terrible: IaC for QEMU/KVM

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) ![Terraform Version](https://img.shields.io/badge/Terraform-v0.12-yellowgreen) ![Ansible Version](https://img.shields.io/badge/Ansible-v2.9%2B-yellowgreen) ![terraform_provider_libvirt](https://img.shields.io/badge/Terraform%20Provider%20Libvirt-v0.6-yellowgreen) ![Made_with_love](https://img.shields.io/badge/made%20with-%E2%9D%A4%EF%B8%8E-red)

![logo](./pics/logo.png)

### Workflows Status

[![Ansible-Lint](https://github.com/89luca89/terrible/workflows/Ansible%20Lint/badge.svg)](https://github.com/89luca89/terrible/actions?query=workflow%3A%22Ansible+Lint%22)
[![Terrible Validate](https://github.com/89luca89/terrible/workflows/Ansible%20Validate/badge.svg)](https://github.com/89luca89/terrible/actions?query=workflow%3A%22Ansible+Validate%22)
[![Ansible Run](https://github.com/89luca89/terrible/actions/workflows/ansible-run.yml/badge.svg)](https://github.com/89luca89/terrible/actions/workflows/ansible-run.yml)

This **Ansible** playbook allows you to initialize and then deploy an entire infrastructure through the aid of **Terraform**, on a **QEMU/KVM** environment.

**Terr**aform + Ans**ible**

## Table of Contents

1. [Abstract](#abstract)
   - [Why not just use a single tool?](#why-not-just-use-a-single-tool)
   - [How it works](#how-it-works)
2. [Requirements](#requirements)
3. [Configuration](#configuration)
   - [Variables](#variables)
   - [Terraform node, Bastions & Jumphosts](#terraform-node-bastions--jumphosts)
   - [Network](#network)
   - [Storage](#storage)
4. [Compatibility](#compatibility)
5. [Installation](#installation)
   - [Container image](#container-image)
6. [Usage](#usage)
   - [Outputs](#outputs)
7. [Authors](#authors)
8. [License](#license)

## Abstract

The Infrastructure as Code had considerable growth in the cloud lately.
However, a separate discussion has to be done regarding the private cloud.
For various reasons, companies may need to use an internal infrastructure instead of cloud ones but,
unfortunately, there aren't as many solutions fsuitable or the private cloud that most of the companies needs.
Our main idea was to implement a flexible and powerful solution to build an infrastructure from scratch effortlessly.
Using the Ansible flexibility and the Terraform power allows users to abstract both the creation and the provisioning of the entire infrastructure,
describing the whole process in a single file, easy to read and to maintain on the long run.

### Why not just use a single tool?

By our side, the need to create a single source of truth for both the infrastructure creation and provisioning, lead to that choice.
As said, our objective was achievable thanks to the Ansible flexibility combined to Jinja2 simplicity that makes us capable of dynamically generating HCL files in order to leverage Terraform power and compatibility to build the infrastructure in no time.

### How It Works

The baisc idea comes from the complexity to automate VMs deployments in a QEMU/KVM environment.

For this reason we decided to automate the deployment process as much as possible, using **Ansible** and **Jinja2**.

First of all we provided a basic **HCL** file (templated with **Jinja2**) describing a basic VM implementation. This is what is usually called IaC (Infrastructure as Code).

Then by using Terraform and its amazing libvirt provider (https://github.com/dmacvicar/terraform-provider-libvirt), we can finally deploy the resultants HCL files generated by Ansible.

The figure below describes the process in an easier way.

![![kvm-deployment]()](./pics/kvm-deployment.jpg)

As you can see, we start with the templated file (`terraform-vm.tf.j2`).

When Ansible runs, it generates *n* `.tf` files, depending on the VM specified inside the inventory. This is the result of the initialization phase.
Once this task finished, the files are completed and ready to be used by Terraform.

At this time, Ansible takes those files and uses Terraform for each instance of them.
Once the task is finished, the VMs previously described into the inventory, will be correctly deployed into the QEMU/KVM server(s).

## Requirements

| Dependency   |      Minimum Version      | Reference |
|----------|:-------------:|-------------|
| Ansible |  2.9+ | https://docs.ansible.com/ |
| Terraform |    0.12+   | https://www.terraform.io/docs/index.html |
| Terraform Provider Libvirt | 0.6+ | https://github.com/dmacvicar/terraform-provider-libvirt/ |
| Libvirt (on the target) | 1.2.14+ | https://libvirt.org/docs.html |

## Configuration

First of all you have to compose the inventory file in a right way. That means you have to describe the VMs you want to deploy into the server.

As you will see, there are some interesting variables you are allowed to use to properly describe your infrastructure.
Some of them are `required` and some others are `optional`.

Below you can see the basic structure for the inventory.

```yaml
all:
    vars:
        ...
    hosts:
        terraform_node:
            ...
        hypervisor_1:
            ...
        hypervisor_2:
            ...
    children:
        deploy:
            vars:
                pool_name: ...
                ...
            children:
                group_1:
                    hosts:
                        host_1:
                            ...
                    vars:
                        ...
                group_2:
                    hosts:
                        host_2:
                            ...
                        host_3:
                            ...
                group_3:
                    hosts:
                        host_4:
                            ...
                    vars:
                        ...
                group_4:
                    hosts:
                        host_5:
                            ...
```

Under the 1st `vars` tag, you can specify the various hypervisors you want to use to distribute your infrastructure.

Here's a little example:

```yaml
all:
    hosts:
        terraform_node:
            ansible_host: 127.0.0.1
            ansible_connection: local
    vars:
        ...
    children:
        deploy:
            vars:
                pool_name: default
        ...
```

In the above example, we specified the *uri* of the QEMU/KVM server (which is going to be common among all the VMs in this specific hypervisor group),
the storage *pool name* for the QEMU/KVM server and the `terraform node` address, which describes where Terraform is installed and where is going to being run.

Now, for each VM we want to specify some property such as the number of the cpu(s), ram, network interfaces, etc.

Here's a little example:

```yaml
        ...
all:
    hosts:
        terraform_node:
            ansible_host: 127.0.0.1
            ansible_connection: local
        hypervisor_1:
            ansible_host: 127.0.0.1
            ansible_connection: local
    children:
        deploy:
            vars:
                pool_name: default
                disk_source: "~/VirtualMachines/centos8-terraform.qcow2"
            children:
                group_1:
                    hosts:
                        host_1:
                            os_family: RedHat
                            cpu: 4
                            memory: 8192
                            hypervisor: hypervisor_1
                            network_interfaces:
                                ...
                group_2:
                    hosts:
                        host_2:
                            os_family: RedHat
                            cpu: 2
                            hypervisor: hypervisor_1
                        host_3:
                            os_family: Suse
                            disk_source: "~/VirtualMachines/opensuse15.2-terraform.qcow2"
                            cpu: 4
                            memory: 4096
                            set_new_passowrd: password123
                            hypervisor: hypervisor_1
                            network_interfaces:
                                ...
```

In this example, we specified 2 main groups (`group_1`, `group_2`) and linked the VMs to `hypervisor_1`.
Those groups are made of 3 VMs (`host_1`, `host_2`, `host_3`).
As you can see, not all the properties has been specified for each machine. This is posible due to the default variables value provided by this playbook.

Thanks to the variables hierarchy in Ansible, you can configure variables:

- Hypervisor wise
- VM group wise
- Single VM wise

This will make easier to manage large homogeneous clusters, still retaining the power of per-VM customization.

![Workflow](pics/workflow.png)

In the above example, we can see, for `hypervisor_1`, the default OS for the VMs is **Centos**, but we specified a different one for `host_3` a single **OpenSuse** node. Similarly for the `hypervisor_2`, the default OS for the VMs is **Ubuntu**, but we specified **Centos** for the `host_4` node.

This kind of configuration granularity is valid for any variable in the playbook.

You can check the default values under `default/main.yml`.

### Variables

Once understood how to fill the inventory file, you are ready to check all the available variables to generate your infrastructure.

These variables are **required**:

* **ansible_host:** `required`. Specifies the ip address for the VM. If not specified, a random ip is assigned.
* **ansible_jump_hosts:** `required` if **terraform_bastion_enabled** is `True`. Specifies one or more jumphost/bastions for the ansible provisioning part.
* **cloud_init**: `optional`. Specifies if the VM uses a cloud-init image or not. `False` If not specified.
* **data_disks:** `optional`. Specifies additional disks to be added to the VM. Check disks section for internal required varibles: [HERE](#storage)
* **disk_source:** `required`. Specifies the (local) path to the virtual disk you want to use to deploy the VMs.
* **hypervisor:** `required`. Specifies on which hypervisor to deploy the Infrastructure.
* **network_interfaces**: `required`. Specifies VM's network interfaces. Check network section for internal required variables: [HERE](#network)
* **os_family:** `required`. Specifies the OS family for the installation. Possible values are: `RedHat`, `Debian`, `Suse`, `Alpine`, `FreeBSD`.
* **pool_name:** `required`. Specifies the *storage pool* name where you want to deploy the VMs on the QEMU/KVM server.
* **ssh_password:** `required`. Specifies the password to access the deployed VMs.
* **ssh_port:** `required`. Specifies the port to access the deployed VMs.
* **ssh_public_key_file:** `required`. Specifies the ssh public key file to deploy on the VMs.
* **ssh_user:** `required`. Specifies the user to access the deployed VMs.

Ansible hosts required outside the `deploy` group:

* **terraform_node:** `required`. Specifies the the machine that performs the Terraform tasks.
* **hypvervisor_[0-9+]:** `required`. Specifies the machine/machines that works as QEMU/KVM hypervisor (at least one machine needed).
The default value of 127.0.0.1 indicates that the machine that perform the Terraform tasks is the same that runs the Ansible playbook. In case the Terraform machine is not the localhost, you can specify the ip/hostname of the Terraform node. More details could be found here: [HERE](#terraform-node-bastions--jumphosts)

These variables are optional, there are sensible defaults set up, most of them can be declared from <ins>hypervisor scope</ins> to <ins>vm-group scope</ins> and <ins>per-vm scope</ins>:

* **change_passwd_command:** `optional`. Specifies a different command to be used to change the user's password. If not specified the default command is used. Default: `echo root:{{ set_new_password }} | chpasswd`. This variable become really useful when you are using a FreeBSD OS.
* **cpu:** `optional`. Specifies the cpu number for the VM. If not specified, the default value is taken. Default: `1`
* **memory:** `optional`. Specifies the memory ram for the VM. If not specified, the default value is taken. Default: `1024`
* **set_new_password:** `optional`. Specifies a new password to access the Vm. If not specified, the default value (**ssh_password**) is taken.
* **terraform_custom_provisioners**: `optional`. Specifies custom shell commands to run on newly created instances BEFORE ansible starts setting them up. Default `""`
* **terrible_custom_provisioners**: `optional`.  Specifies custom shell commands to run AFTER terrible run is completed (for example calling specific `ansible pull` for each node). Default `""`
* **vm_autoboot**: `optional`. Specifies if the VM should be automatically started at boot. Default: `False`
* **base_deploy_path**: `optional`. Specifies where the Terraform files and state will be deployed, the default value is `$HOME`
* **state_save_file**: `optional`. Specifies where the output terrible state is stored, the default value is `PATH_TO_THE_INVENTORY-state.tar.gz`


### Terraform Node, Bastions & Jumphosts

The following section will describe some different deployment scenarios that we may tipically encounter.

As described above, the `terraform_node` variable is `required`.
The `terraform_node` could be local or remote.

A really common scenario will have a local Terraform node. This could be declared as follows:

```yaml
all:
    vars:
        ...
    hosts:
        terraform_node:
            ansible_host: 127.0.0.1
            ansible_connection: local
        hypervisor_1:
            ansible_host: 127.0.0.1
            ansible_connection: local
```

This scenario assumes that the `terraform_node` is the same host that is running the Ansible playbook.
This case will ask Terraform to connect to QEMU/KVM using the uri `qemu:///system`.

![local-all](./pics/local-all.png)

---

If you want to use a remote QEMU/KVM server instead, you can do this:

```yaml
all:
    vars:
        ...
    hosts:
        terraform_node:
          ansible_host: 127.0.0.1
          ansible_connection: local
        hypervisor_1:
          ansible_host: remote_kvm_machine.domain
          ansible_user: root
          ansible_port: 22
          ansible_ssh_pass: password
```
This case will ask Terraform to connect to the QEMU/KVM server using the following uri: `qemu+ssh://root@remote_kvm_machine.domain/system`.
This also setup the Terraform internal ssh connection to use it as a **bastion** host to connect to his VMs.

![remote-kvm](./pics/remote-kvm.png)

---

Also, Terraform could be separated from Ansible and be located on a remote server.
You can declare it simply by using the `ansible_host` variable, as follows:

```yaml
all:
    vars:
        ...
    hosts:
        terraform_node:
          ansible_host: remote_terraform_node.domain
          ansible_connection: ssh # or paramiko or whatever NOT local
        hypervisor_1:
          ansible_host: remote_terraform_node.domain
          ansible_connection: ssh # or paramiko or whatever NOT local
          ansible_user: root
          ansible_port: 22
          ansible_ssh_pass: password
```
This assumes that the `terraform_node` is the same host that is running the QEMU/KVM hypervisor.
This case will ask Terraform to connect to QEMU/KVM using the uri `qemu:///system`.
The *post-deployment* task of this Ansible playbook, has to use a **jumphost** to get access to the VMs of the internal network. For this reason we need to use the `terraform_node` as jumphost to reach them.

![remote-terraform](./pics/remote-terraform.png)

---

Also, if you have a remote QEMU/KVM server and a remote Terraform server, you can use them as follows:

```yaml
all:
    vars:
        ...
    hosts:
        terraform_node:
          ansible_host: remote_terraform_node.test.com
          ansible_connection: ssh # or paramiko or whatever NOT local
        hypervisor_1:
          ansible_host: remote_kvm_machine.domain
          ansible_user: root
          ansible_port: 22
          ansible_ssh_pass: password
```
This case will ask Terraform to connect to QEMU/KVM using the uri: `qemu+ssh://root@remote_kvm_machine.domain/system`.
This also setups the Terraform internal ssh connection to use it as a bastion to connect to its VMs.
Since already remote, this case will set up 2 jumphosts for Ansible, first one is the `terraform_node`, the other
one is the `ansible_host`.

![remote-all](./pics/remote-all.png)


### Network

Network declaration is <ins>mandatory and per-VM</ins>.

Declare each device you want to add inside the `network_interfaces` dictionary.

**Be aware that:**

- <ins>the order of declaration is important</ins>
- the NAT device should <ins>always be present</ins> (unless you can control your DHCP leases
    for the external devices) and that should be the <ins>first</ins> device.
    It's an important parameter for the way the playbook has to communicate
    with the VM <ins>before</ins> setting up all the userspace networks.
- the `default_route` should be assigned to <ins>one</ins> interface to function properly. If not set it's equal to False.

Supported interface types:

* `nat`
* `macvtap`
* `bridge`


Structure:

```yaml
        all:
            vars:
                pool_name: default
                disk_source: "~/VirtualMachines/centos8-terraform.qcow2"
            hosts:
                terraform_node:
                    ansible_host: 127.0.0.1
                    ansible_connection: local
                hypervisor_1:
                    ansible_host: 127.0.0.1
                    ansible_connection: local
            children:
                group_1:
                    hosts:
                        host_1:
                            ansible_host: 172.16.0.155
                            os_family: RedHat
                            cpu: 4
                            memory: 8192
                            hypervisor: hypervisor_1
                            network_interfaces:
                                # Nat interface, it should always be the first one you declare.
                                # it does not necessary have to be your default_route or main ansible_host,
                                # but it's important to declare it so ansible has a way to communicate with
                                # the VM and setup all the remaining networks.
                                iface_1:
                                  name: nat             # mandatory
                                  type: nat             # mandatory
                                  ip: 192.168.122.47    # mandatory
                                  gw: 192.168.122.1     # mandatory
                                  dns:                  # mandatory
                                   - 1.1.1.1
                                   - 8.8.8.8
                                  mac_address: "AA:BB:CC:11:24:68"   # optional
                                  # default_route: False
                                iface_2:
                                  name: ens1p0      # mandatory
                                  type: macvtap     # mandatory
                                  ip: 172.16.0.155  # mandatory
                                  gw: 172.16.0.1    # mandatory
                                  dns:              # optional
                                   - 1.1.1.1
                                   - 8.8.8.8
                                  default_route: True # at least one true mandatory, false is optional.
```

Variables explanation:

* **name:** `required` Specifies the name for the connection, this is important for `bridge` and `macvtap` types as it will be the interface/bridge on the host on which they will be created.
* **type:** `required` Specifies interface type, supported types are **nat**, **macvtap**, **bridge**.
* **ip:** `required` Specifies the IP to assign to this interface.
* **gw:** `required` Specifies the Gateway of this interface.
* **default_route:** `at least one required` Specifies if this interface is the default route or not. **At least one interface set as True**.
* **dns:** `required` Specifies the dns list for this interface, this is an array of IPs.
* **mac_address:** `optional` Specifies the mac address for this interface.


The playbook will use the available IP returned from the `terraform apply` command to access the machines
and use the `os_family` way to setup the user-space part of the network:

- static IPs
- routes
- gateways
- DNS

After that, the playbook will set the `ansible_host` variable to its original value, and proceed with
the provisioning.

This is important because it will make `ansible_host` independent from the internal management interface
needed for this network bootstrap tasks, making it easily compatible with any type of role that you
want to perform after that.

During this process, virtual networks (bridges, VLANs, etc...) **inside the vm** will be ignored, this will improve detection
of the networks we want to manage, improving compatibility with docker, k8s, nested-virtualization, etc...

### Storage

This section explain how to add additional disks to the VMs.

Suppose that you want to create a VM that needs a large amount of storage, and a separated disk just to store the configurations. This will be quite simple to achieve.

The main variable you need is `data_disks`, then you have to specify the disks and the related properties for each one.

If `data_disks` is mentioned in your inventory, the following variables are required:

* **size:** `required`. Specifies the disk size expressed in GB. (eg. `size: 1` means 1GB)
* **pool:** `required`. Specifies the pool where you want to store the additional disks.
* **format:** `require`. Specifies the filesystem format you want to apply to the disk. Available filesystems are specified below.
* **mount_point:** `required`. Specifies the mount point you want to create for the disk, use `none` if declaring a swap disk.
* **encryption:** `required`. Specifies the mount point of the disk. Available values could be `True` or `False`.

​	**N.B.** Each disk declared must have a unique name (eg. you can't use `disk0` twice).

| OS Family   |  Supported Disk Format      |  Encryption Supported  |
|----------|:-------------|--------------|
| Debian |  `ext2`, `ext3`, `ext4`, `swap` | yes |
| Alpine |  `ext2`, `ext3`, `ext4`, `swap` | yes |
| FreeBSD |  `ufs`, `swap`     |  no  |
| RedHat | `ext2`, `ext3`, `ext4`, `xfs`, `swap` | yes |
| Suse | `ext2`, `ext3`, `ext4`, `xfs`, `swap` | yes |

Let's take a look at how the *inventory* file is going to be fill.

```yaml
        all:
            vars:
                pool_name: default
                disk_source: "~/VirtualMachines/centos8-terraform.qcow2"
            hosts:
                terraform_node:
                    ansible_host: 127.0.0.1
                    ansible_connection: local
                hypervisor_1:
                    ansible_host: 127.0.0.1
                    ansible_connection: local
            children:
                group_1:
                    hosts:
                        host_1:
                            ansible_host: 172.16.0.155
                            os_family: RedHat
                            cpu: 4
                            memory: 8192
                            hypervisor: hypervisor_1
                            # Here we start to declare
                            # the additional disk.
                            data_disks:
                                # Here we declare the disk name
                            	disk-storage: disk0                 # Uniqe name to identify the disk unit.
                            		size: 100                       # Disk size = 100 GB
                            		pool: default                   # Store the disk image into the pool = default.
                            		format: xfs                     # Disk Filesystem = xfs
                            		mount_point: /mnt/data_storage  # The path where the disk is mounted, none is using swap
                            		encryption: True                # Enable disk encryption

                                # Here we declare the disk name
                            	disk-swap: swp0                     # Uniqe name to identify the disk unit.
                            		size: 1                         # Disk size = 1 GB
                            		pool: default                   # Store the disk image into the pool = default.
                            		format: swap                    # Disk Filesystem = swap
                            		mount_point: none               # The path where the disk is mounted, none if using swap
                            		encryption: False               # Does not enable disk encryption
```



## Compatibility

At this time the playbook supports the most common 4 OS families for the Guests:

* Alpine
* RedHat
    * RedHat7
    * RedHat8
    * Centos7
    * Centos8
    * RockyLinux
    * Almalinux
    * Fedora and derivatives ( **untested** )
* Debian
    * Debian 9
    * Debian 10
    * Ubuntu 18
    * Ubuntu 20
    * Other derivatives ( **untested** )
* Suse
    * Leap
    * Thumbleweed
    * Other derivatives ( **untested** )
* FreeBSD
    * FreeBSD 12.x
    * FreeBSD 13.x
    * Other derivatives ( **untested** )

This means you'll be able to generate the infrastructure using **ONLY** the OS listed above.

Hypervisor OS is agnostic, as long as requirements are met.

## Installation

Before using **Terrible**, the following system dependencies needs to be installed: [dependencies](#requirements).

Use the following command to satisfy the project dependencies:

```bash
pip3 install --user -r requirements.txt
```

### Container Image

To avoid boring dependencies installation to make Terrible works and speed up your infrastructure deployment, we provided a `Dockerfile` to build yourself a minimal image with all you need.

We use a `debian:buster-slim` image to have a compact system, fully compatibile with all the required tools. The container image uses the latest Terrible tag's release.

The minimum packages required to run the container image is **Docker (or Podman)** and **QEMU/KVM** installed on the system.

#### Pull

If you are a lazy person (just like us), you can directly pull the latest image release from DockerHub.

```bash
docker pull 89luca89/terrible:latest
```

#### Build

To build the image, instead, type the following command:

```bash
docker build -t terrible .
```

This will take some time, grab a cup of coffee and wait.

#### Run

Once you've built the image, you're ready to run it as in the example below:

```bash
docker run \
    -it \
    --rm \
    -v /var/run/libvirt/libvirt-sock:/var/run/libvirt/libvirt-sock \
    -v ./inventory-test.yml:/terrible/inventory-test.yml \
    -v /path/to/vm/images/:/opt/ \
    -v ~/.ssh/:/root/.ssh/ \
    89luca89/terrible
```

**N.B.** If you are using RHEL, CentOS or Fedora you need to add the `--privileged` flag because otherwise SELinux does not allow it to access the libvirt socket.

**N.B.** If you are using your local QEMU/KVM instance, remember to add `--net=host` to the command.

**Notes:**

* The volume `/var/run/libvirt/libvirt-sock` is mandatory if you want to run Terrible locally (a local QEMU/KVM instance). In this way you'll directly interact with the QEMU/KVM api, provided by the system.

* The volume `./inventory-test.yml` has to include the inventory file inside the container, to deploy the infrastructure.

* The volume `~/VirtualMachines/` has to include the `qcow2` images inside the container to deploy them.
* The volume `~/.ssh` has to include your ssh keys into the container to deploy them inside the infrastructure machines.

## Usage

To speed up your deployment process, we made our Packer template files available.

The other repository could be found here: [packer-terraform-kvm](https://github.com/89luca89/packer-terraform-kvm).

Once composed the inventory file, it's time to run your playbook.

To pull up infrastructure:

```bash
ansible-playbook -i inventory.yml -u root main.yml
```

To validate the inventory file:

```bash
ansible-playbook -i inventory.yml -u root main.yml --tags validate
```

To pull down infrastructure (maintaining the resources in place):

```bash
ansible-playbook -i inventory.yml -u root main.yml --tags destroy
```

To completely delete the infrastructure:

```bash
ansible-playbook -i inventory.yml -u root main.yml --tags purge
```

### Outputs

Based on your inventory, the complete state of the infrastructure (TF files, TF states, LUKS keys etc...)
will be written to `${INVENTORY_NAME}-state.tar.gz` this file is essential to keep track of the
infrastructure state.

You can (and should) save this state file to keep track of the infrastructure complete state, the *.tar.gz
will be restored and saved on each run.

## Authors

- Luca Di Maio      <luca.dimaio1@gmail.com>
- Alessio Greggi    <greggialessio@gmail.com>

## License

- GNU GPLv3, See LICENSE file.
