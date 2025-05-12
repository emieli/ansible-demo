# Chapter 6: Trunk switchports & STP root bridge
To show what more Ansible can do, let's implement some more features. One such important feature is making FW-SW and SW-SW interfaces into trunk switchports. There's little point creating VLANs if your switchports won't forward the traffic. Another "important" feature is deciding which switch should be Spanning-Tree root bridge. All switches have the same priority by default, so the switch MAC-address is typically what decides who becomes STP root. To force SW-1 (who is closest to the firewall) to become STP root bridge, we need to make some config changes.

# STP Root Bridge
We will use LLDP to decide which switch should become STP root bridge, specifically by checking if the switch has FW-1 as a LLDP neighbor. If they are neighbors, we make that switch STP root. We have to do this in a few steps:

## 1. Enable LLDP on Fortigate
I have created two new roles that are run in the order shown below.

**playbook_arista_vlans_add.yml**:
```yaml
- name: "Fortigate"
  hosts: FW-1
  tasks:
    - include_role:
        name: fortios_enable_lldp

- name: "Arista"
  hosts: "EOS"
  tasks:
    - include_role:
        name: arista_stp_root
```

First we enable LLDP on the Fortigate interface, let's see what that looks like.

**roles/fortios_enable_lldp/tasks/main.yml**:
```yaml
---
- name: "Fortigate: Enable LLDP"
  fortinet.fortios.fortios_system_interface:
    vdom: "root"
    state: "present"
    system_interface:
      name: "port2"
      lldp_transmission: enable
  register: result

# Only pause if LLDP was not already enabled
- ansible.builtin.pause:
    seconds: 5
  when: result.changed
```

We have two tasks. The first tasks enables LLDP transmissions on port2. We save the result to variable **result**. The second task checks if the first task completed with status **changed** and if so, it pauses for a few seconds. This is to allow the LLDP process to send out an LLDP packet to SW-1 before the playbook continues. We don't want the playbook to pause if LLDP is already enabled, hence the **when: result.changed**.

That's all on the Fortigate.

## 2. Make one switch STP root
The second role in our playbook is **arista_stp_root**, ensuring that the switch directly connected to our firewall become STP root. Let's examine it below.

**roles/arista_stp_root/tasks/main.yml**:
```yaml
---
- name: "Get LLDP neighbors"
  arista.eos.eos_command:
    commands:
      - command: show lldp neighbors
        output: json
  register: lldp_neighbors

### OUTPUT ###
# ok: [SW-1] =>
#   lldp_neighbors:
#     changed: false
#     failed: false
#     stdout:
#     - lldpNeighbors:
#       - neighborDevice: fw-1
#         neighborPort: port2
#         port: Ethernet1
#         ttl: 120
#       - neighborDevice: sw-2
#         neighborPort: Ethernet1
#         port: Ethernet2
#         ttl: 120

- name: "Set STP root"
  arista.eos.eos_config:
    lines: spanning-tree mst 0 priority 8192
  loop: "{{ lldp_neighbors.stdout.0.lldpNeighbors }}"
  loop_control:
    loop_var: nbr
  when: '"fw-" in nbr.neighborDevice'
```

First we fetch LLDP neighbors and save the output to variable **lldp_neighbors**. In the second task we apply the line **spanning-tree mst 0 priority 8192**, but only if one of our neighbors has **fw-** in its name. Priority 8192 is better than the default 32768, making it pretty much guaranteed that this switch will become STP root.

Alright, this is going great. On to the next task!

# Trunk switchports
Just like we used LLDP to find the switch connected to our firewall, we will use LLDP to figure out which switchports to turn into trunk-ports. Any port with a **fw-** or **sw-** neighbor should become a trunk port. Let's add it to the playbook.

**playbook_arista_vlans_add.yml**:
```yaml
[...]

- name: "Arista"
  hosts: "EOS"
  tasks:
    - include_role:
        name: arista_trunk_ports
```

We have added the **arista_trunk_ports** role to the playbook. Its contents are shown below.

**roles/arista_trunk_ports/tasks/main.yml**:
```yaml
---
- name: "Get LLDP neighbors"
  arista.eos.eos_command:
    commands:
      - command: show lldp neighbors
        output: json
  register: lldp_neighbors

### OUTPUT ###
# ok: [SW-1] =>
#   lldp_neighbors:
#     changed: false
#     failed: false
#     stdout:
#     - lldpNeighbors:
#       - neighborDevice: fw-1
#         neighborPort: port2
#         port: Ethernet1
#         ttl: 120
#       - neighborDevice: sw-2
#         neighborPort: Ethernet1
#         port: Ethernet2
#         ttl: 120
#       - neighborDevice: sw-2
#         neighborPort: Management0
#         port: Management0
#         ttl: 120

# Make FW-SW and SW-SW trunk ports
- name: "Configure trunk ports"
  arista.eos.eos_config:
    lines: switchport mode trunk
    parents: interface {{ nbr.port }}
  loop: "{{ lldp_neighbors.stdout.0.lldpNeighbors }}"
  loop_control:
    loop_var: nbr
  when:
    - '"Ethernet" in nbr.port'
    - '"fw-" in nbr.neighborDevice or "sw-" in nbr.neighborDevice'
```

Just like last time, we fetch LLDP neighbors and save the output to **lldp_neighbors**. We then loop through all LLDP neighbors. A switchport becomes a trunk-port if it contains "Ethernet" AND has a **fw-** or **sw-** neighbor. As an exercise for you, why are both **when** statements required? What happens if we omit the first one?

<details>
  <summary>Answer</summary>
  The output above show that we get sw-2 as LLDP neighbor on the Ma0 port, which is strictly used for management. To avoid our management-port becoming a trunk-port, we need the "Ethernet in nbr.port" in our when statement. 
</details>

# Don't repeat yourself (DRY)
Both roles **arista_stp_root** and **arista_trunk_ports** run the same "Get LLDP neighbors" code block that produces exactly the same output. There is a programming principle called DRY proclaiming that duplicated code is a sign of poor design. Each copy must be maintained over time which is extra work and, some say, extra complexity. This is partly why Ansible Roles exist, to avoid unnecessary code duplication. I would like to weigh in on this topic.

My personal take is that DRY is overrated. Duplicated code is typically only a problem if you see yourself writing the same thing 3-4+ times. I much prefer the principle of "Locality of behavior" which recommends to "keep related code close together". For example, if we look at the playbook, all it says is that it runs the two roles. There is no information to indicate that these roles rely on a variable called **lldp_neighbors**.

We could "fix" the duplication by creating a third **arista_lldp_neighbors** role and ensure that we run it first. This makes the **lldp_neighbors** variable accessible to the roles that follow it. This fixes the problem, but creates another. We now have a hidden dependency between these three roles, making the order these roles run very important. If we come back in six months and start reordering these roles, we might break the playbook without realizing it.  
This is the reason why I have the "Get LLDP neighbors" task in both roles. If we need this information inside the role, it's best to fetch the information as part of the job that role performs. This make the role easy to understand while minimizing its coupling to other parts of our code-base. We can change the format of **lldp_neighbors** in one role without it affecting the other role. Yes, we do the same thing twice, but it's not an expensive operation and the benefits is easier-to-read code with few dependencies. 

Ok, rant over. While this may feel like an overkill topic for you, it's likely that you will find yourself thinking about these things when your Ansible code-base grows. 

# Run the playbook
Yes. I have been talking for long enough now. Let's run the playbook. 

**playbook_arista_vlans_add.yml**:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_add.yml

PLAY [Fortigate] 

TASK [include_role : fortios_enable_lldp] 
included: fortios_enable_lldp for FW-1

TASK [fortios_enable_lldp : Fortigate: Enable LLDP] 
changed: [FW-1]

TASK [fortios_enable_lldp : Wait for LLDP packet to be sent] 
Pausing for 5 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [FW-1]

TASK [Add vlan HERP] 
ok: [FW-1]

TASK [Add vlan DERP] 
ok: [FW-1]

PLAY [Arista] 

TASK [Add vlans] 
ok: [SW-1]
ok: [SW-2]

TASK [include_role : arista_trunk_ports] 
included: arista_trunk_ports for SW-1, SW-2

TASK [arista_trunk_ports : Get LLDP neighbors] 
ok: [SW-2]
ok: [SW-1]

TASK [arista_trunk_ports : Configure trunk ports] 
changed: [SW-2] => (item={'port': 'Ethernet1', 'neighborDevice': 'sw-1', 'neighborPort': 'Ethernet2', 'ttl': 120})
changed: [SW-1] => (item={'port': 'Ethernet1', 'neighborDevice': 'fw-1', 'neighborPort': 'port2', 'ttl': 120})
skipping: [SW-2] => (item={'port': 'Management0', 'neighborDevice': 'sw-1', 'neighborPort': 'Management0', 'ttl': 120})
changed: [SW-1] => (item={'port': 'Ethernet2', 'neighborDevice': 'sw-2', 'neighborPort': 'Ethernet1', 'ttl': 120})
skipping: [SW-1] => (item={'port': 'Management0', 'neighborDevice': 'sw-2', 'neighborPort': 'Management0', 'ttl': 120})

TASK [include_role : arista_stp_root] 
included: arista_stp_root for SW-1, SW-2

TASK [arista_stp_root : Get LLDP neighbors] 
ok: [SW-1]
ok: [SW-2]

TASK [arista_stp_root : Set STP root] 
skipping: [SW-2] => (item={'port': 'Ethernet1', 'neighborDevice': 'sw-1', 'neighborPort': 'Ethernet2', 'ttl': 120})
skipping: [SW-2] => (item={'port': 'Management0', 'neighborDevice': 'sw-1', 'neighborPort': 'Management0', 'ttl': 120})
skipping: [SW-2]
changed: [SW-1] => (item={'port': 'Ethernet1', 'neighborDevice': 'fw-1', 'neighborPort': 'port2', 'ttl': 120})
skipping: [SW-1] => (item={'port': 'Ethernet2', 'neighborDevice': 'sw-2', 'neighborPort': 'Ethernet1', 'ttl': 120})
skipping: [SW-1] => (item={'port': 'Management0', 'neighborDevice': 'sw-2', 'neighborPort': 'Management0', 'ttl': 120})

PLAY RECAP 
FW-1                       : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
SW-1                       : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
SW-2                       : ok=6    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

As we can see above, LLDP is enabled on the Fortigate followed by a 5-second pause. Our switches then configure their switchports and SW-1 changes its STP priority to 8192 to make it the root bridge. We can verify the changes by connecting to the switch via SSH (admin/admin):
```
(venv) emileli@clab:~/ansible-demo$ ssh admin@172.20.20.3
(admin@172.20.20.3) Password:
sw-1>ena
sw-1#show lldp neighbors
Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           fw-1                     port2               120
Et2           sw-2                     Ethernet1           120

sw-1#show spanning-tree
  This bridge is the root
  Bridge ID  Priority 8192

sw-1#show interfaces trunk
Port            Mode            Status          Native vlan
Et1             trunk           trunking        1
Et2             trunk           trunking        1

Port            Vlans in spanning tree forwarding state
Et1             1-3
Et2             1-3
```

That's it for this chapter! We focused on fetching data and using **loops** with **when** statements to make strategic configuration decisions. Thanks for reading.
