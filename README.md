# Deploy servers on Exoscale and configure their private network interface

## Requirements

First get access to the playbooks by cloning the repository:

    $ git clone git@github.com:marcaurele/ansible-exoscale-privnet.git
    $ cd ansible-exoscale-privnet

All the needed tools (Ansible and cs) are listed in the requirements.txt file and you can install them with a simple:

    $ pip install -r requirements.txt

As per cs [documentation](https://github.com/exoscale/cs), export the following values in your shell:

```
CLOUDSTACK_ENDPOINT="https://api.exoscale.ch/compute"
CLOUDSTACK_KEY="your api key"
CLOUDSTACK_SECRET_KEY="your secret key"
```


Now, to run the playbook we use the `ansible-playbook` command:

    $ ansible-playbook create-instances-playbook.yml

