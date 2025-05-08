# Chapter 5: Configure Fortigate subinterfaces
Just creating the VLANs on the switches typically isn't good enough. The devices on that VLAN need some way to exit their local subnet and talk to other devices on your network. We need a default gateway. This is where our lovely Fortigate firewall comes. It will act as default gateway and perform inter-vlan routing while doing some of that sweet security-enchancing stateful inspection.

I have taken the liberty of extending the playbook to include some Fortigate tasks. Let's review them:

**playbook_arista_vlans_add.yml**:
```yaml
- name: "Arista"
  hosts: "EOS"
  tasks:
    - name: "Add vlans"
      arista.eos.eos_vlans:
        state: merged
        config:
          - name: HERP
            vlan_id: 2
          - name: DERP
            vlan_id: 3

- name: "Fortigate"
  hosts: FORTIOS
  tasks:
    - name: "Add vlan HERP"
      fortinet.fortios.fortios_system_interface:
        vdom: "root"
        state: "present"
        system_interface:
          allowaccess:
            - "ping"
          interface: "port2"
          ip: "10.1.2.1/24"
          name: "HERP"
          type: "vlan"
          vlanid: 2

    - name: "Add vlan DERP"
      fortinet.fortios.fortios_system_interface:
        vdom: "root"
        state: "present"
        system_interface:
          allowaccess:
            - "ping"
          interface: "port2"
          ip: "10.1.3.1/24"
          name: "HERP"
          type: "vlan"
          vlanid: 3
```

The playbook now adds two subinterfaces to our Fortigate firewall: HERP and DERP. HERP (vlan 2) is configured with IP-address 10.1.2.1/24 and DERP (vlan 3) is configured with IP-address 10.1.3.1/24. The two subinterfaces are attached to the physical **port2** interface which connects to **SW-1**. The **present** state provides idempotency, ensuring the interfaces exist. 

Let's run the playbook to apply:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml

PLAY [Arista] 

TASK [Add vlans] 
ok: [SW-2]
ok: [SW-1]

PLAY [Fortigate] 

TASK [Add vlan HERP] 
fatal: [FW-1]: UNREACHABLE! =>
    changed: false
    msg: 'Failed to connect to the host via ssh: ansible@172.20.20.4: Permission denied
        (publickey,password).'
    unreachable: true

PLAY RECAP 
FW-1                       : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
SW-1                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Ha! Did you really think it would be that easy? Ansible defaults to SSH connectivity, which **fortinet.fortios** doesn't support. So we need to tell Ansible to use HTTPAPI, just like we did with our Arista switches. Let's update our inventory config, but this time we spice it up by creating a **group_vars** folder and placing the config inside of the **FORTIOS.yml** file.

**group_vars/FORTIOS.yml**:
```yaml
ansible_connection: ansible.netcommon.httpapi
ansible_network_os: fortinet.fortios.fortios
ansible_httpapi_use_ssl: false
ansible_httpapi_validate_certs: false
```
*We have to set **use_ssl** to false as I'm using a trial VM with limited cryptographic features*

I'll also paste the **hosts.yml** for reference, although no changes should be necessary here:
```yaml
SITE01:
  hosts:
    FW-1:
      ansible_host: 172.20.20.4
    SW-1:
      ansible_host: 172.20.20.3
    SW-2:
      ansible_host: 172.20.20.2
EOS:
  hosts:
    SW-1:
    SW-2:
  vars:
    ansible_connection: ansible.netcommon.httpapi
    ansible_network_os: arista.eos.eos
    ansible_httpapi_use_ssl: false
    ansible_httpapi_validate_certs: false
    ansible_user: admin
    ansible_password: admin
    ansible_become: true
FORTIOS:
  hosts:
    FW-1:
```

We can verify that the settings have been applied correctly with this command:
```json
(venv) emileli@clab:~/ansible-demo$ ansible-inventory --host FW-1
{
    "ansible_connection": "ansible.netcommon.httpapi",
    "ansible_host": "172.20.20.4",
    "ansible_httpapi_session_key": {
        "access_token": "4tg5qc1b7tGjhc6gmwjfhgbw1gdc68"
    },
    "ansible_httpapi_use_ssl": false, 
    "ansible_httpapi_validate_certs": false,
    "ansible_network_os": "fortinet.fortios.fortios",
    "ansible_user": "ansible"
}
```


If your config looks like this, you can run the playbook again:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml

PLAY [Arista] 

TASK [Add vlans] 
ok: [SW-2]
ok: [SW-1]

PLAY [Fortigate] 

TASK [Add vlan HERP] 
changed: [FW-1]

TASK [Add vlan DERP] 
changed: [FW-1]

PLAY RECAP 
FW-1                       : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-1                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Success! We now have subinterfaces to match our VLANs! If you want to verify, feel free to ssh to the Fortigate and run **show system interface** and **get router info routing-table connected** to see that the interfaces exist and are active. 

# Limit
The hosts used in this playbook are EOS and FORTIOS which include all switches/firewalls, respectively. Our inventory only contain a single **SITE01** location, but imagine we added a **SITE02** that contained FW-2, SW-3 and SW-4 (yes, the names are stupid). If we were to then run this playbook, our HERP and DERP vlans would be created on both sites. You may want this on the switches, but we probably don't want to create the **10.1.3.0/24** subnet on multiple sites. 

The simplest solution to this problem is to add the **--limit SITE01** to the end of your playbook as shown below. This ensures that the playbook will only run on EOS/FORTIOS devices that belong to the **SITE01** location. 
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml --limit SITE01
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml --limit SITE01 --limit FORTIOS
```
*The second line show that multiple limits can be added. In this case the playbook will only run FW-1 tasks.*

# The end?
You have reached the end of my tutorial. I hope you're enjoyed walking through it as much as I enjoyed making it. I hope this helps you get started on your Ansible-journey and that you have an idea of something in your daily work that can be automated.
