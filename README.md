*Enter chapter-8 branch*
```
git switch chapter-8
```

# Chapter 8: Pass variables into playbook
So far in the lab we have been hardcoding values in our playbooks. For example, our vlans HERP and DERP have both had their name and ID hardcoded. This chapter will explore a more realistic approach where you, the user, supply the Name and ID of the VLAN when you start the playbook. This makes the playbook reusable as adding new VLANs won't require actually changing the playbooks themselves.

The way we achieve this is with so-called **--extra-vars variable=value** syntax that inject extra variables and values to the **ansible-playbook** command. A shorter way of writing the same thing is with the **-e** flag. I have created a new **playbook_vlan_add.yml** that expects the user to provide an **id** variable with a value, aswell as a **name** variable with a value.

In the example below, I want to create a vlan named GUEST with a vlan ID of 10. This is the playbook output:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_vlan_add.yml -e name=GUEST -e id=10 -l SITE01
[WARNING]: Found variable using reserved name: name

PLAY [Input validation] 

TASK [Validate vlan ID] 
ok: [localhost]

TASK [Validate vlan name] 
ok: [localhost]

PLAY [Add VLAN] 

TASK [include_role : vlan_add] 
included: vlan_add for FW-1, SW-1, SW-2

TASK [vlan_add : include_tasks] 
included: /home/emileli/ansible-demo/roles/vlan_add/tasks/fortios.yml for FW-1
included: /home/emileli/ansible-demo/roles/vlan_add/tasks/eos.yml for SW-1, SW-2

TASK [vlan_add : Add vlan 10 - GUEST] 
changed: [FW-1]

TASK [vlan_add : Add vlan 10 - GUEST] 
changed: [SW-1]
changed: [SW-2]

PLAY RECAP 
FW-1                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-1                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The playbook above execute two Plays: **Input validation** and **Add VLAN**. Let's look at them one at a time.

# Input validation
As we are now asking the user to supply information, we need to make sure the input they provide is valid. For example, if they attempt to add vlan 5000 (maximum vlan is 4095) then the playbook should abort with an error. Another constraint is the name of the VLAN, where Fortigate don't allow interface names longer than 15 characters. We should verify these things before we allow the playbook to continue. I do input validation for both the **id** and **name** variables:

## Vlan ID validation

**playbook_vlan_add.yml**:
```yaml
- name: Input validation
  hosts: localhost
  tasks:
    - name: "Validate vlan ID"
      ansible.builtin.assert:
        that:
          - id is defined
          - id | int < 4095
          - id | int > 1
        msg: |
          Vlan argument 'id' missing or not within 2-4094 range.
          Example: ansible-playbook playbooks/vlan_add.yml -e id=123
        quiet: true
      any_errors_fatal: true
```

I'm running the play locally on **localhost** as it makes sense to do the validation before we open a connection to our remote devices. We use the builtin **assert** module to make sure that a variable **id** is defined and that its value is between 2 and 4094. If it's not, we display the **msg** and abort the playbook with **any_errors_fatal: true**. 

If vlan ID validation was successful, we move on to the vlan name:

## Vlan name validation

**playbook_vlan_add.yml**:
```yaml
- name: Input validation
  hosts: localhost
  tasks:
    - name: "Validate vlan name"
      ansible.builtin.assert:
        that:
          - name is defined
          - name | length > 0
          - name | length < 16
        msg: |
          Vlan argument 'name' missing, empty or too long.
          Fortigate interface names must be 15 characters or less.
          Example: ansible-playbook playbooks/vlan_add.yml -e name=FLERP
        quiet: true
      any_errors_fatal: true
```

The assert statement is almost identical. We ensure that the variable exist and that its length is between 1 and 15 characters. If not, show error message and abort playbook.

# Add vlan
Now that input has been validated and the playbook has green light to continue, we fire off the second play.

**playbook_vlan_add.yml**:
```yaml
- name: "Add VLAN"
  hosts:
    - FORTIOS
    - EOS
  tasks:
    - include_role:
        name: vlan_add
```

The play contains a new role **vlan_add**. What's new about this role is that it includes both FORTIOS and EOS tasks. Let's see what files the new role include:
```
(venv) emileli@clab:~/ansible-demo$ tree roles/
roles/
└── vlan_add
    └── tasks
        ├── eos.yml
        ├── fortios.yml
        └── main.yml
```

As we know from previous chapters, Ansible automatically look for the **main.yml** file, so let's view its contents first:

**roles/vlan_add/tasks/main.yml**:
```yaml
- ansible.builtin.include_tasks: "{{ ansible_network_os.split('.')[-1] }}.yml"
```

Wow, that was short. It calls the **include_tasks** module to run a task-file. We generate the name of the file dynamically by taking the **ansible_network_os** value (fortinet.fortios.**fortios** or arista.eos.**eos**).

We then split the string on every dot and only keep the last part that was split. When **FW-1** enter this stage, the **fortios.yml** task-file is run. When **SW-1** or **SW-2** enter this stage, the **eos.yml** task-file is run. Let's examine each file.

**roles/vlan_add/tasks/fortios.yml**:
```yaml
- name: "Add vlan {{ id }} - {{ name }}"
  fortinet.fortios.fortios_system_interface:
    vdom: "root"
    state: "present"
    system_interface:
      allowaccess:
        - "ping"
      interface: "port2"
      ip: "10.1.{{ id }}.1/24"
      name: "{{ name }}"
      type: "vlan"
      vlanid: "{{ id }}"
      vdom: "root"
```

The fortios task-file includes a single task that create a subinterface based on our playbook input. We can see the **id** and **name** variables referenced inside curly brackets, allowing their respective values to be inserted. The **id** variable is also used to generate the third subnet octet. Can you find an issue with this? 

The main interface **port2** is still hardcoded, but the playbook is still much more dynamic than it was before.

Let's examine the eos task-file:

**roles/vlan_add/tasks/eos.yml**:
```yaml
- name: "Add vlan {{ id }} - {{ name }}"
  arista.eos.eos_vlans:
    state: merged
    config:
      - name: "{{ name }}"
        vlan_id: "{{ id }}"
```

The same is true here. We reference the **id** and **name** variables to dynamically add the VLAN to our switches.

# What's the point?
My goal with this chapter is to show that a role can include multiple files, making it easy to extend as new network devices are added to your network. By having **main.yml** split tasks into separate files (based on network OS in this case) we can safely edit the task-file for one device type without risk of impacting others. 

While my example above is very small, this separation become important as your project grow over time. Having a good structure from the start can help avoid headaches in the future.

# Exercise
Feel free to play around with the playbook, adding a few new VLANs and subnets. My playbook isn't perfect, I bet you can find one or two ways to break it. If you find a way, see if you can patch the playbook to make it not break.

# The end
This is the end of the Ansible workshop. Great job to you for getting through the whole thing! I hope it was instructive and worth your time. Bookmark this GIT repository to use as a future reference. Please let me know if you found any issues while going through the chapters.

Resources
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#defining-variables-at-runtime
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_tasks_module.html
