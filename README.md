# Chapter 2: First Playbook
I have prepared a **playbook_arista_vlans_show.yml** for us, let's run it:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans]
fatal: [SW-1]: FAILED! =>
    changed: false
    msg: Connection type ssh is not valid for this module
fatal: [SW-2]: FAILED! =>
    changed: false
    msg: Connection type ssh is not valid for this module

PLAY RECAP 
SW-1                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
SW-2                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

That didn't go well. Ansible tried to use SSH to connect to the switch, but the SSH connection type is not supported by the Arista Ansible modules. Instead, it expects Ansible to communicate via Rest API (HTTPS). Let's fix this by editing our inventory file to look like this:

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
EOS:
  hosts:
    SW-1:
    SW-2:
  vars:
    ansible_connection: ansible.netcommon.httpapi
    ansible_network_os: arista.eos.eos
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
```

The above config creates a new group (EOS) and adds SW-1 and SW-2 as members. We then tell Ansible to use its HTTP-API module when communicating with these Arista switches. Ansible also need to know what OS the switches are, which we supply with the **ansible_network_os** line. Finally we tell Ansible to use HTTPS but not validate the certificate, as the Arista API certificate is self-signed.

Let's run the playbook again and see what happens:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans]
fatal: [SW-1]: FAILED! =>
    changed: false
    msg: 'HTTP Error 401: Unauthorized'
fatal: [SW-2]: FAILED! =>
    changed: false
    msg: 'HTTP Error 401: Unauthorized'

PLAY RECAP 
SW-1                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
SW-2                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

Progress! Yes, the playbook still failed, but atleast we got a different error message. It's not clear what credentials Ansible are using to connect, so make sure it's using the correct ones. Open up hosts.yml again and make it look like this:

**hosts:yml**:
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
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
    ansible_user: admin
    ansible_password: admin
```

Ansible now know to use the admin/admin credentials when connection.

Ok, let's try again:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans]
ok: [SW-1]
ok: [SW-2]

TASK [Arista: Show output: vlans] 
ok: [SW-2] =>
    vlans:
        changed: false
        failed: false
        gathered: []
ok: [SW-1] =>
    vlans:
        changed: false
        failed: false
        gathered: []

TASK [Arista: Show output: vlans.gathered] 
ok: [SW-1] =>
    vlans.gathered: []
ok: [SW-2] =>
    vlans.gathered: []

PLAY RECAP 
SW-1                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Success! All tasks were completed. Let us take a moment to look at the generated output and compare it to the actual playbook below. A playbook contains a list of "Plays" to perform. In this playbook there is only a single play: "Show Arista VLANs". It's configured to run on two hosts, SW-1 and SW-2.

Our play contain three tasks. The first task uses the **eos_vlans** module which gather information about the VLANs configured on the switch. The result is registered in a variable named **vlans**.  The following two tasks are two **debug** tasks where the data fetched from the sswitch is printed to the Ansible playbook output.

I chose to use two debug statements to illustrate how Ansible can "drill down" into variables to fetch specific values. For example, the first debug statement prints the full **vlans** output, which contains three key-value pairs: **changed**, **failed**, **gathered**. Since we only fetched data, nothing was changed. Fetching the data went well, there was no failure in doing so. Finally, the gathered key returned an empty list. We can see that it is a list by the **[]**, and we can see that it is empty because there is no text between the brackets.

In the second debug statement we only print the **vlans.gathered** output, giving us the empty list. Drilling down like this is a very powerful feature, allowing Ansible to react to data that is fetched. In this example we only print the data that was returned, but Ansible can do more.

**playbook_arista_vlans_show.yml**:
```yaml
---
- name: Show Arista VLANs
  hosts:
    - "SW-1"
    - "SW-2"
  tasks:

    - name: "Arista: Get vlans"
      arista.eos.eos_vlans:
        state: gathered
      register: vlans

    - name: "Arista: Show output: vlans"
      debug:
        var: vlans

    - name: "Arista: Show output: vlans.gathered"
      debug:
        var: vlans.gathered
```

Next up is chapter 3:
```
git switch chapter-3
```

Resources:
> https://docs.ansible.com/ansible/latest/collections/arista/eos/eos_vlans_module.html
