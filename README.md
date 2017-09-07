# Ansible playbook to deploy a DHCP server for private networking on Exoscale


On [Exoscale](https://www.exoscale.ch), the [private
network](https://community.exoscale.ch/documentation/compute/privnet/) you can
attach to your VM comes raw. To ease the communication between your instances
on this private link, the easiest is to setup a DHCP server answering requests
on the private networking interface for all your VMs using this network. Don't
forget that Security Groups don't apply there, if you want to harden the
security.

We are going to use Ansible to ease the process of deploying one DHCP server
and 3 sample virtual machines to validate the setup.

If you're not familiar with [Ansible](https://www.ansible.com), it's an open
source automation tools for your infrastructure as a code written in Python.
The only language you'll need is
[Yaml](https://docs.ansible.com/ansible/latest/YAMLSyntax.html) to write the
configuration files.

## Setup

First we will clone the repository:

    $ git clone git@github.com:marcaurele/ansible-exoscale-privnet.git
    $ cd ansible-exoscale-privnet

Create a new [virtual environment for python](https://virtualenv.pypa.io),
preferably with Python 2.7 since Ansible is not fully compatible with
Python 3.x:

    $ virtualenv -p <location_of_python_2.7> venv
    # Activate the virtual environment
    $ . ./venv/bin/activate

Install the requirements for the playbook ([ansible](https://pypi.python.org/pypi/ansible) *(>=2.4)*, [cs](https://pypi.python.org/pypi/cs), [sshpubkeys](https://pypi.python.org/pypi/sshpubkeys)):

    $ pip install -r requirements.txt

In your shell you need to export those 3 variables, as per
[cs documentation](https://github.com/exoscale/cs/):
  - `export CLOUDSTACK_ENDPOINT=https://api.exoscale.ch/compute`
  - `export CLOUDSTACK_KEY=<your-api-key>`
  - `export CLOUDSTACK_SECRET=<your-api-secret-key>`

Or if you're alread using a `.cloudstack.ini` file, you only need to export:
  - `export CLOUDSTACK_REGION=<section_name>`

*Your API key can be found at https://portal.exoscale.ch/account/profile/api.*

Now we are all set to run the playbook. To verify the setup, from your
terminal run:

    $ cs listZones

```bash
› cs listZones
{
  "count": 3, 
  "zone": [
    {
      "allocationstate": "Enabled", 
      "dhcpprovider": "VirtualRouter", 
      "id": "1128bd56-b4d9-4ac6-a7b9-c715b187ce11", 
      "localstorageenabled": true, 
      "name": "ch-gva-2", 
      "networktype": "Basic", 
      "securitygroupsenabled": true, 
      "tags": [], 
      "zonetoken": "ccb0a60c-79c8-3230-ab8b-8bdbe8c45bb7"
    }, 
    {
      "allocationstate": "Enabled", 
      "dhcpprovider": "VirtualRouter", 
      "id": "91e5e9e4-c9ed-4b76-bee4-427004b3baf9", 
      "localstorageenabled": true, 
      "name": "ch-dk-2", 
      "networktype": "Basic", 
      "securitygroupsenabled": true, 
      "tags": [], 
      "zonetoken": "fe63f9cb-ff75-31d3-8c46-3631f7fcd533"
    }, 
    {
      "allocationstate": "Enabled", 
      "dhcpprovider": "VirtualRouter", 
      "id": "4da1b188-dcd6-4ff5-b7fd-bde984055548", 
      "localstorageenabled": true, 
      "name": "at-vie-1", 
      "networktype": "Basic", 
      "securitygroupsenabled": true, 
      "tags": [], 
      "zonetoken": "26d84c22-f66d-377e-93ab-987ef477cab3"
    }
  ]
}
```
You should get a JSON output of the current zones available on Exoscale. If
it worked, perfect you're set. If not, please check again the steps.

## Quick run

We will discuss the playbook setup after. If you're eager to run the playbook
and see the result, run:

    $ ansible-playbook deploy-privnet-dhcp.yml

## Playbook roles

### Infra

This role provisions the VMs on Exoscale, as well as a new SSH key, security
groups and add the private networking interface to each of them. What you
might not see often is the user data provided for the VM deployment in
[create_vm.yml](https://github.com/marcaurele/ansible-exoscale-privnet/roles/infra/tasks/create_vm.yml)
which is use to let cloud-init manage the hostname nicely as done when starting
a VM from Exsocale portal:

```yaml
    user_data: |
      #cloud-config
      manage_etc_hosts: true
      fqdn: {{ dhcp_name }}
```

The private networking interface is added through
[create_private_nic.yml](https://github.com/marcaurele/ansible-exoscale-privnet/roles/infra/tasks/create_private_nic.yml)
using Ansible module [`cs_instance_nic`](http://docs.ansible.com/ansible/latest/cs_instance_nic_module.html)

```yaml
- name: "dhcp server : add privnet nic"
  local_action:
    module: cs_instance_nic
    network: privNetForBasicZone
    vm: "{{ dhcp_name }}"
    zone: "{{ zone }}"
```

This says: attach a new NIC to my vm `{{ dhcp_name }}` in the zone `{{ zone }}`
on the network named `privNetForBasicZone`. This new interface will come up
in your Linux Ubuntu box as `ens7`, which is the interface on which the DHCP
server will listen.

### DHCP/server

This role configure the DHCP server instance. This one must have a static IP
configured for its privnet interface `ens7` using for example the gateway IP
send on to clients if you would want to use it as a router. In
[configure_private_nic.yml](https://github.com/marcaurele/ansible-exoscale-privnet/roles/dhcp/server/tasks/configure_private_nic.yml)
we upload the static configuration and activate the interface after:

```yaml
- name: "dhcp server : add privnet nic"
  local_action:
    module: cs_instance_nic
    network: privNetForBasicZone
    vm: "{{ dhcp_name }}"
    zone: "{{ zone }}"

- name: "sample vms : add privnet nic to {{ num_nodes }} vms"
  local_action:
    module: cs_instance_nic
    network: privNetForBasicZone
    vm: "privnet-{{ item }}"
    zone: "{{ zone }}"
  with_sequence: count={{ num_nodes }}
  when: num_nodes is defined
```

### DHCP/client

This role is the simplest through [configure_private_nic.yml](https://github.com/marcaurele/ansible-exoscale-privnet/blob/master/roles/dhcp/client/tasks/configure_private_nic.yml),
it uploads the [network interface configuration
file](https://github.com/marcaurele/ansible-exoscale-privnet/blob/master/roles/dhcp/client/files/privnet.cfg) for the privnet and enables it:

```yaml
- name: copy network interface configuration
  copy:
    src: privnet.cfg
    dest: /etc/network/interfaces.d/01-privnet.cfg
    force: yes

- name: enable privnet interface
  shell: "ifup ens7"
```
