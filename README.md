# Chapter 1: Installing Ansible and creating Inventory file
Ansible is written in Python, so installing Ansible is as simple as installing the Ansible Python package:
```
emileli@clab:~/ansible-demo$ python3 -m venv venv
emileli@clab:~/ansible-demo$ source venv/bin/activate
(venv) emileli@clab:~/ansible-demo$ pip install --upgrade pip
(venv) emileli@clab:~/ansible-demo$ pip install ansible
(venv) emileli@clab:~/ansible-demo$ ansible --version
ansible [core 2.18.5]
```

To give some deeper explanation, we first create a new Python Virtual Environment (venv). This creates a local Python environment in your ansible-demo folder. We then use the **source** command to enter the environment before installing the Ansible package. By installing packages in the venv we keep them local to this folder. So if we have multiple python projects running, each project has its own set of packages that won't conflict with other projects. This is a common best practice that is useful to follow.

To show that the binary changes, we can run these commands:
```
(venv) emileli@clab:~/ansible-demo$ deactivate
emileli@clab:~/ansible-demo$ which python3
/usr/bin/python3

emileli@clab:~/ansible-demo$ source venv/bin/activate
(venv) emileli@clab:~/ansible-demo$ which python3
/home/emileli/ansible-demo/venv/bin/python3
```

Once we have entered our venv, we upgrade PIP (package installer for python) to the latest version so that it can fetch the latest version of ansible for us. Then we finally install ansible. Once the install is complete, we check which version was installed (2.18.5).

If you want to exit the venv, execute the **deactivate** command. You shouldn't have to do this though.

## Enter chapter-1 GIT branch
Before we continue with this chapter, we need to enter the correct GIT branch. This guide is serving each chapter in its own branch. To access the chapter-1 branch, enter the **git switch chapter-1** command:
```
emileli@clab:~/ansible-demo$ git switch chapter-1
branch 'chapter-1' set up to track 'origin/chapter-1'.
Switched to a new branch 'chapter-1'
emileli@clab:~/ansible-demo$
```
*We are now in the chapter-1 branch*

With this short detour completed, let's continue with the actual chapter contents!

# Create Ansible Inventory file
Now that Ansible is installed, we need to build our Ansible inventory so that it can interact with FW-1, SW-1 and SW-2. Feel free to do this in vim or nano. The file should be named **hosts.yml**. 

**hosts.yml**:
```yaml
SITE01:
  hosts:
    FW-1:
      ansible_host: 172.20.20.4
    SW-1:
      ansible_host: 172.20.20.3
    SW-2:
      ansible_host: 172.20.20.2
```
*run **clab inspect** to get management IP-addresses*

We now have to tell Ansible which file to look in, so we have to create an ansible configuration file and make it look like below.

**ansible.cfg**:
```ini
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
```json
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

Next up is chapter 2: https://github.com/emieli/ansible-demo/tree/chapter-2
