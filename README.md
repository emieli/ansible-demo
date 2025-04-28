# Chapter 1: Installing Ansible and creating Inventory file
Ansible is written in Python, so installing Ansible is as simple as installing the Ansible Python package:
```
$ python3 -m venv venv
$ source venv/bin/activate
(venv) $ pip install --upgrade pip
(venv) $ pip install ansible
(venv) $ ansible --version
ansible [core 2.18.5]
```

To give some deeper explanation, we first create a new Python Virtual Environment (venv). This creates a local python environment in your ansible-demo folder. We then use the **source** command to enter our local environment before install the Ansible package. By installing packages in the venv we keep them local to this folder. So if we have multiple python projects running, each project has its own set of packages that won't conflict with other projects. This is a common best practice that is useful to follow.

Once we have entered our venv, we upgrade PIP (package installer for python) to the latest version so that it can fetch the latest version of ansible for us. Then we finally install ansible. Once the install is complete, we check which version was run.

If you want to exit the venv, enter the **deactivate** command. You shouldn't have to do this though.

## Create Ansible Inventory file
Now that Ansible is installed, we need to build our Ansible inventory so that it can interact with FW-1, SW-1 and SW-2. Feel free to do this in vim or nano. The file should be named **hosts.yml**. 

**hosts.yml**:
```
SITE01:
  hosts:
    FW-1:
      ansible_host: 172.20.20.4
    SW-1:
      ansible_host: 172.20.20.3
    SW-2:
      ansible_host: 172.20.20.2
```

We now have to tell Ansible which file to look in, so we have to create an ansible configuration file and make it look like below.

**ansible.cfg**:
```
[defaults]
inventory = ./hosts.yml

gathering = explicit
host_key_checking = False
callback_result_format = yaml
```

I snuck in a few more lines in the file while we have it open:
- Display results in yaml format. The default fomat is json, which is not as easy to read.
- Default "gathering" state is **implicit** which means the playbook will fetch a lot of data. By making it explcit, the playbooks will not automatically gather any data, making it run much faster.
- Disable SSH host-key checking. Not a best practice, but useful in our lab environment.

We can test that the inventory is loaded properly with the **ansible-inventory --list** command:
```
$ ansible-inventory --list
{
    "SITE01": {
        "hosts": [
            "FW-1",
            "SW-1",
            "SW-2"
        ]
    },
    "_meta": {
        "hostvars": {
            "FW-1": {
                "ansible_host": "172.20.20.4"
            },
            "SW-1": {
                "ansible_host": "172.20.20.3"
            },
            "SW-2": {
                "ansible_host": "172.20.20.2"
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "SITE01"
        ]
    }
}
```

We have now installed Ansible and created a inventory file. Next up is creating our first Ansible playbook!

