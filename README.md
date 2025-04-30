# Chapter 3: Playbook to Add a VLAN
This time a playbook **playbook_arista_vlans_add.yml** has been added. Let's run it:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml

PLAY [Add VLANs] 

TASK [Arista: Add vlans]
changed: [SW-2]
changed: [SW-1]

PLAY RECAP 
SW-1                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Ok, some "Add vlans" task obviously changed something on our two switches. Let's run **playbook_arista_vlans_show.yml** to figure out what that change was:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans]
ok: [SW-1]
ok: [SW-2]

TASK [Arista: Show output: vlans]
ok: [SW-1] =>
    vlans:
        changed: false
        failed: false
        gathered:
        -   name: HERP
            state: active
            vlan_id: 2
        -   name: DERP
            state: active
            vlan_id: 3
ok: [SW-2] =>
    vlans:
        changed: false
        failed: false
        gathered:
        -   name: HERP
            state: active
            vlan_id: 2
        -   name: DERP
            state: active
            vlan_id: 3

TASK [Arista: Show output: vlans.gathered.0] 
ok: [SW-1] =>
    msg: Vlan 2 is named HERP
ok: [SW-2] =>
    msg: Vlan 2 is named HERP

PLAY RECAP 
SW-1                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Ok, so two vlans were added: HERP (2) and DERP (3). Let's look at the playbook to verify that this was indeed the change it wanted to make.

**playbook_arista_vlan_add.yml**:
```yaml
(venv) emileli@clab:~/ansible-demo$ cat playbook_arista_vlans_add.yml
---
- name: Add VLANs
  hosts: "EOS"
  tasks:
    - name: "Arista: Add vlans"
      arista.eos.eos_vlans:
        state: merged
        config:
          - name: HERP
            vlan_id: 2
          - name: DERP
            vlan_id: 3
```

This playbook contains a single "Add VLANs" play, which in turn contains a single **eos_vlans** task. The play is set to only run on hosts belonging to the EOS group, which is SW-1 and SW-2. The **eos_vlans** task is set to **merge** two Vlans into the current configuration, HERP and DERP. 

So far we have seen two states used by the **eos_vlans** task: **gathered** and **merged**. Other states such as **overridden** and **deleted**, also exist but are not covered here. You can find them all in the documentation, just google for the module name. Since we don't want to delete or override any existing config, the **merged** state makes the most sense.

I did add some funny output to the end of the **vlans_show** playbook in this chapter, let's see what it did:
```
TASK [Arista: Show output: vlans.gathered.0] 
ok: [SW-1] =>
    msg: Vlan 2 is named HERP
ok: [SW-2] =>
    msg: Vlan 2 is named HERP
```

Before we look to deeply, let's compare it to the corresponding task "code":
```
- name: "Arista: Show output: vlans.gathered.0"
  debug:
    msg: "Vlan {{ vlans.gathered.0.vlan_id }} is named {{ vlans.gathered.0.name }}"
  when: "vlans.gathered | length > 0"
```

By drilling further into the **vlans** variable, we were able to fetch the name and ID of the first VLAN in the **gathered** list and print it in the playbook output. Entries in a list are zero-indexed, meaning the first entry is 0, the second entry is 1, etc. 
We had to add a **when** statement to the task to make sure it only runs when the list is not empty, as trying to access the first entry in an empty list will cause a playbook runtime error. 

# Idempotency
Ok, switching gears a bit. When we first ran the **vlans_add** playbook, we saw that the output for SW-1 and SW-2 showed the **Changed** state. If we run the playbook again, what output do we get?

```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml

PLAY [Add VLANs] 

TASK [Arista: Add vlans] 
ok: [SW-2]
ok: [SW-1]

PLAY RECAP 
SW-1                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

This time the playbook said **ok** instead of **changed**. Also note that the playbook didn't crash or throw some error, even though it technically didn't do its job. It's job was to add two VLANs to the switches, but it did not do any such thing. 

Although my arguments are a bit contrived here, I'm trying to highlight the purpose of idempotency. We want the playbooks we build to be predictable and consistent. If the end goal is to add a VLAN but the VLAN already exists, the end goal is still achieved. The playbook should be able to run multiple times, producing the same results each time. If the job is to delete a VLAN, but it has already been deleted, the palybook should still run just fine. If your playbook breaks because it failed to delete a non-existent VLAN, it is not idempotent.

This is why the **eos_vlans** module is state-based. We tell it what state we expect, and it makes sure to get us there. Since I can't provide any state in the **debug** tasks, I had to use the **when** statement to enforce idempotency. 
I couldn't be sure that you would run my two playbooks in order while reading this chapter, so I had to make sure that the playbook would not throw an error if you decided to run **vlans_show** before any VLANs had been added to the switch.

That's it for now! Maybe some FortiOS stuff in the next chapter?
```
git switch chapter-4
```
https://github.com/emieli/ansible-demo/tree/chapter-4
