# Deploy on Exoscale and setup a DHCP server for private networking

## Requirements

First get access to the playbooks by cloning the repository:

    $ git clone git@github.com:marcaurele/ansible-exoscale-privnet.git
    $ cd ansible-exoscale-privnet

All the needed tools (Ansible, cs and sshpubkeys) are listed in the requirements.txt file that you can install in a new virtual environment:

    $ pip install -r requirements.txt

As per cs [documentation](https://github.com/exoscale/cs), export the following values in your shell:

```
CLOUDSTACK_ENDPOINT="https://api.exoscale.ch/compute"
CLOUDSTACK_KEY="your api key"
CLOUDSTACK_SECRET_KEY="your secret key"
```

or if you have a `.cloudstack.ini` file you can use `CLOUDSTACK_REGION` environment variable.


Now, to run the playbook we use the `ansible-playbook` command:

    $ ansible-playbook deploy-privnet-dhcp.yml

    # With a section named ' exoscale' in .cloudstack.ini
    $ CLOUDSTACK_REGION=exoscale ansible-playbook deploy-privnet-dhcp.yml

