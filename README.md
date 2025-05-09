# Chapter 4: Generate Fortigate API access-token
We want Ansible to also create subinterfaces for the VLAN's we have been creating so far. We do however immediately run into a problem. The fortigate does not have any HTTPAPI access out of the box, so before we can do any Ansible-automation using the **fortinet.fortios** package, we have to generate an API access-token that Ansible can use.

This output show the manual way to generate the token on your Fortigate:
```
(venv) emileli@clab:~/ansible-demo$ ssh admin@172.20.20.4
admin@172.20.20.4's password:

fw-1 # config system api-user
fw-1 (api-user) # edit ansible
fw-1 (ansible) # set accprofile prof_admin
fw-1 (ansible) # end
fw-1 # execute api-user generate-key ansible

New API key: 38c9tf65gd18jpxr96gQn59dfwnxjg

NOTE: The bearer of this API key will be granted all access privileges assigned to the api-user ansible.

fw-1 # exit
Connection to 172.20.20.4 closed.
```

We create a new **ansible** API-user and give it **prof_admin** privileges. Once created, we generate an API access-token, which we must provide in our API-requests to this Fortigate. This access-token should be finally added to our Ansible inventory file in the **ansible_httpapi_session_key** variable.

# Let's automate it!
But this is an Ansible course after all, we want to automate all the things! I have provided a playbook to do this job for us. Before we run it, let's examine its contents:

**playbook_generate_fortios_access_token.yml**:
```yaml
- name: "Localhost"
  hosts: localhost
  tasks:
    - include_role:
        name: generate_fortios_access_token
      loop: "{{ groups.FORTIOS }}"
      loop_control:
        loop_var: fgt
      when: hostvars[fgt]['ansible_httpapi_session_key'] is undefined      
```

We are introducing a few concepts here. This playbook contains a single play named "Localhost". The name is pretty bad, but I want to emphasize that this playbook is meant to be run locally on this server, not on any remote device. We can see this is indeed the case with the **hosts: localhost** line. The reason for this will become apparent later.

Our play contains a single task, which is to run an Ansible role named **generate_fortios_access_token**. We can probably guess what it does by the name. The purpose of an Ansible role is to create reusable code blocks that can be referenced by many playbooks. This avoids code duplication as we can keep the role in one place and just point to them in our playbooks. Another reason is that it's easier to run a role while in a loop, but now I'm getting ahead of myself.

The third new concept is the **loop**. To make sure that we generate API access-tokens for all Fortigates in our environemtn, I use the **groups.FORTIOS** variable to get a list of all Fortigates that belong to the FORTIOS group in our inventory file. Right now there's only a single Fortigate (FW-1), but by using a loop we seamlessly allow newly added Fortigates to also have an access-token generated.  
Let's assume that **groups.FORTIOS** contains the following list: **["FW-1", "FW-2"]**. On the first iteration of the loop, variable **fgt** gets the value **FW-1**. The variable is sent along when the role **generate_fortios_access_token** is then run, allowing tasks inside that role to access **fgt** and retrieve its **FW-1** value. Once the role has run all its tasks, the loop iterates and **fgt** now has the **FW-2** value, and the role is run again. 

Finally, we use the **when is undefined** statement to not generate an token on a Fortigate if a token already exist. The playbook achieves this by having the loop check the **when** statement before running the role. If the **when** statement is false, this iteration of the loop is skipped.  
In practice, this means that we can now run the playbook as many times as we want. Only the first time the playbook runs on a Fortigate will a token be generated. This is very powerful as we can now add new Fortigates to our inventory and ensure that they have API-tokens applied while also being certain that existing fortigates will not have their token changed unnecessarily.

# Roles
So we know that the role exist somewhere in our ansible code base. Where does it live? Let's take a look:

```
(venv) emileli@clab:~/ansible-demo$ tree -I venv -I clab*
.
├── ansible.cfg
├── hosts.yml
├── playbook_arista_vlans_add.yml
├── playbook_arista_vlans_show.yml
├── playbook_generate_fortios_access_token.yml
├── README.md
├── roles
│   └── generate_fortios_access_token
│       └── tasks
│           └── main.yml
```

A new **roles** folder has magically appeared. Inside it lives a second folder named **generate_fortios_access_token** which seems to perfectly match the name of the **include_role** in our playbook, a weird coincidence? This is exactly how Ansible finds what role to run, by looking for a folder with the same name inside the **roles** folder. Ansible also automatically looks for a **tasks** subfolder, and automatically looks for a **main.yml** for tasks to run. This is the folder structure that Ansible has designed, and one that we should follow in order to build scalable Ansible projects. 

# Generating the API access-token
Let us examine the main.yml file.

**roles/generate_fortios_access_token/tasks/main.yml**:
```yaml
- name: "Create {{ fgt }} API user and generate access-token"
  ansible.builtin.shell: |
    set timeout 30
    spawn ssh admin@{{ hostvars[fgt]['ansible_host'] }}
    expect {
      "This key is not known by any other names" {
        send "yes\n"
        exp_continue
      }
      "password: " {
        send "admin\n"
      }
    }
    expect "fw-1 "
    send "config system api-user\n"
    expect "fw-1 "
    send "edit ansible\n"
    expect "fw-1 "
    send "set accprofile prof_admin\n"
    expect "fw-1 "
     send "end\n"
    expect "fw-1 "
    send "execute api-user generate-key ansible\n"
    expect "fw-1 "
    send "exit\n"
  args:
    executable: /usr/bin/expect
  register: output

- name: "Create access_token variable from 'NEW API key' line output"
  set_fact:
    access_token:  "{{ item.split()[-1] }}"
  when: '"New API key" in item'
  loop: "{{ output.stdout_lines }}"

- name: "Print {{ fgt }} access token"
  debug:
    var: access_token

- name: "Create host_vars/ folder"
  ansible.builtin.file:
    path: "host_vars"
    state: directory

- name: "Generate host_vars/{{ fgt }}.yml"
  ansible.builtin.blockinfile:
    create: true
    path: host_vars/{{ fgt }}.yml
    block: |
      ansible_user: ansible
      ansible_httpapi_session_key:
        access_token: {{ access_token }}
```

The **main.yml** file contain a list of tasks that should be run. These tasks are all run on **localhost**, which is our linux VM. The first task is an **ansible.builtin.shell** that specifically run the **usr/bin/except** binary. The point of **expect** is to run certain commands in a shell, and based on the expected output run other commands.

We are telling our local linux VM to SSH to a server and run a bunch of commands on it. In our case we open a SSH session to some device via the **admin@{{ hostvars[fgt]['ansible_host'] }}** command. This introduces another concept that we haven't touched on yet. Apart from the **groups** variable that we saw earlier, Ansible also provides a **hostvars** variable for use in our playbooks. The hostvars variable contain all hostvars information, including information that we provided in the inventory file.

In short, we can access the management IP-address of FW-1 by looking at **hostvars["FW-1"]['ansible_host']**. By wrapping this variable in **{{ }}** curly brackets, we tell Ansible to replace **hostvars[...** with 172.20.20.4. So the spawn command effectively becomes **spawn ssh admin@172.20.20.4**. The curly brackets are used in Jinja templates to explain that the value of the variable inside the {{ }} should be printed in the template. Ansible makes heavy use of Jinja, although we haven't really looked that deep into it yet.

> **_NOTE:_** While I am glossing over the whole hostvars thing, feel free to run **playbook_show_hostvars.yml** to see what the hostvars variable contains. It's a very useful variable. 

### First task
connect to Fortigate via SSH, run some commands to create API-user **ansible** and generate an API access-token. All output is registered to the **output** variable in Ansible, mainly so that we can capture the access-token and do something with it.

### Second task
we loop through all lines of the registered output, searching for a line containing the "NEW API key" output. Once that line is found, we split the line every time we encounter a whitespace and fetch the last entry as our access-token. We save the access-token to variable **access_token**.
The syntax may be very confusing. You will see the loop in action when you run the playbook later, so hopefully it will make more sense then.

### Third task
Print the access-token to screen, just to make sure that we did indeed fetch the token and nothing else.

### Fourth and fifth task
Finally, we create a new **host_vars/** folder, writing username and access-token to the **host_vars/FW-1.yml** file. This is what the file looks like after the playbook has run:

**host_vars/FW-1.yml**:
```yaml
# BEGIN ANSIBLE MANAGED BLOCK
ansible_user: ansible
ansible_httpapi_session_key:
  access_token: 94Nctmy6gjr6fNfcfNrxGg0ncz5xzn
# END ANSIBLE MANAGED BLOCK
```

Let's breathe. It is OK if you need to read this section one more time. What the playbook does is connect to the Fortigate via SSH, run some commands to generate an API-user and access-token. The playbook is recording all output for us, allowing us to capture the access-token from the output and save it to a file. Still confused? Go back and read it again. Feel ok? Continue onwards :)

# Folder host_vars
Wait what, we're putting inventory stuff in a separate folder? Yep! This is yet another design decision by Ansible to allow your inventory to scale. Having all inventory information in a single **hosts.yml** file will not scale once your network grows to hundreds or even thousands of network devices. Splitting it up into folders and per-device files help keep the inventory manageable. 
The folder must be named **host_vars** and the filename must match the name of the node (case sensitive) with a **.yml** after to show that it's a yaml file. 

Likewise, a **group_vars** folder can be created where a file exist for each group. We could use a **group_vars/EOS.yml** file, for example.

There is an order of precedence for where Ansible prefer to fetch data as there may be multiple sources. I was unable to find the documentation link describing this behavior, but if you accidentally create the same key in two different files but stick two different values in them, one of the values will be overriding the other. Just something to keep in mind.

# Can we run it already?
Ok, I've explained the new playbook enough. Let's actually run it. 
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_generate_fortios_access_token.yml

PLAY [Localhost] 

TASK [include_role : generate_fortios_access_token] 
included: generate_fortios_access_token for localhost => (item=FW-1)

TASK [generate_fortios_access_token : Create FW-1 API user and generate access-token] 
changed: [localhost]

TASK [generate_fortios_access_token : Create access_token variable from 'NEW API key' line output] 
skipping: [localhost] => (item=spawn ssh admin@172.20.20.4)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=admin@172.20.20.4's password: )
skipping: [localhost] => (item=client_input_hostkeys: convert key: Invalid key length)
skipping: [localhost] => (item=fw-1 # config system api-user)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=fw-1 (api-user) # edit ansible)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=fw-1 (ansible) # set accprofile prof_admin)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=fw-1 (ansible) # end)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=fw-1 # execute api-user generate-key ansible)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=)
ok: [localhost] => (item=New API key: 7zhm31qHfqtnHd00NzfbQ01c34nbk8)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=NOTE: The bearer of this API key will be granted all access privileges assigned to the api-user ansible.)
skipping: [localhost] => (item=)
skipping: [localhost] => (item=fw-1 # )

TASK [generate_fortios_access_token : Print FW-1 access token] 
ok: [localhost] =>
    access_token: 7zhm31qHfqtnHd00NzfbQ01c34nbk8

TASK [generate_fortios_access_token : Create host_vars/ folder] 
ok: [localhost]

TASK [generate_fortios_access_token : Generate host_vars/FW-1.yml] 
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************
localhost                  : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

We can see the loop going through each line of output, skipping lines until it finds one where the text **NEW API Key** is present. We can then see that the access-token is extracted. The changes are saved to **host_vars/FW-1.yml**, just as discussed previously.

Just to show the power of the **when is undefined** statement earlier, let's run it again:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_generate_fortios_access_token.yml

PLAY [Localhost] 

TASK [include_role : generate_fortios_access_token] 
skipping: [localhost] => (item=FW-1)
skipping: [localhost]

PLAY RECAP 
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```
*FW-1 is skipped as the token already exist. Very nice!*

Let's run two commands to verify that the playbook actually did anything:
```
(venv) emileli@clab:~/ansible-demo$ cat host_vars/FW-1.yml
# BEGIN ANSIBLE MANAGED BLOCK
ansible_user: ansible
ansible_httpapi_session_key:
  access_token: 7zhm31qHfqtnHd00NzfbQ01c34nbk8
# END ANSIBLE MANAGED BLOCK

(venv) emileli@clab:~/ansible-demo$ ansible-inventory --host FW-1 --yaml
ansible_host: 172.20.20.4
ansible_httpapi_session_key:
  access_token: 7zhm31qHfqtnHd00NzfbQ01c34nbk8
ansible_user: ansible

```

We are now ready to use the Ansible module **fortinet.fortios** to Automate our Fortigate config, onward to the next chapter!

```
git switch chapter-5
```
https://github.com/emieli/ansible-demo/tree/chapter-5

Resources:
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/blockinfile_module.html
