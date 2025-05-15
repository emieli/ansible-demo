# Welcome!
This is an Ansible workshop, helping you learn how to get started with Ansible and its features.

## Chapters
This workshop contain eight chapters. We start nice and easy, introducing concepts like inventory, hostvars and groupvars. We then read through and run a few playbooks to understand the syntax better. We show ways to scale up your project with roles. A few chapters in, you're asked to build your own playbook, then break one that I built. We also review Ansible Vault, encrypting sensitive data.

## Requirements
This workshop was made for colleagues at my company where I provided a containerlab environment. If you stumble across this repository, you won't be able to run the playbooks unless you setup a containerlab yourself. You can find the topology file in this repo, but I do not provide the images. Both the Arista and Fortinet images are free to download though, you just need to register a free account. Try sticking with 7.0.x Fortigate images as later release trains use a highly restricted trial mode.

# Chapter 0: Getting started
In this chapter we deploy our lab environment so that we have something to run ansible commands against. We'll be setting up a Fortigate Firewall and two Arista cEOS L3 switches. Arista CLI is very similar to Cisco syntax. In later chapters we'll be configuring VLANs through Ansible playbooks. 

## Topology
![topology](ansible-demo.svg)

## Step 1
SSH to our Linux server deployed for this purpose:
```
ssh 10.215.200.25
```
*Contact emileli if login fails*

## Step 2
Clone this git repo in your home folder, creating a "ansible-demo" directory:
```
cd && git clone https://github.com/emieli/ansible-demo.git ansible-demo
cd ansible-demo/
```

## Step 3
Deploy your containerlab topology:
```
echo "name: $USER" >> topology.clab.yml
containerlab deploy
```
*This will take a minute to run, as the nodes take some time to get started*

When the deployment is done, you will see output in this format:
```
╭───────────────────┬────────────────────────────┬─────────┬───────────────────╮
│        Name       │         Kind/Image         │  State  │   IPv4/6 Address  │
├───────────────────┼────────────────────────────┼─────────┼───────────────────┤
│ clab-emileli-fw-1 │ fortinet_fortigate         │ running │ 172.20.20.3       │
│                   │ vrnetlab/vr-fortios:7.0.17 │         │ 3fff:172:20:20::3 │
├───────────────────┼────────────────────────────┼─────────┼───────────────────┤
│ clab-emileli-sw-1 │ arista_ceos                │ running │ 172.20.20.4       │
│                   │ ceos:4.33.2F               │         │ 3fff:172:20:20::4 │
├───────────────────┼────────────────────────────┼─────────┼───────────────────┤
│ clab-emileli-sw-2 │ arista_ceos                │ running │ 172.20.20.2       │
│                   │ ceos:4.33.2F               │         │ 3fff:172:20:20::2 │
╰───────────────────┴────────────────────────────┴─────────┴───────────────────╯
```
*You can get this view again with the **clab inspect** command*

If you want, you can try SSH:ing to one of the nodes. Default login is admin/admin:
```
emileli@clab:~/ansible-demo$ ssh admin@172.20.20.4
(admin@172.20.20.4) Password:
sw-1>ena
sw-1#sh ip int brief
                                                                              Address
Interface         IP Address           Status       Protocol           MTU    Owner
----------------- -------------------- ------------ -------------- ---------- -------
Management0       172.20.20.4/24       up           up                1500

sw-1#sh int desc
Interface                      Status         Protocol           Description
Et1                            up             up
Et2                            up             up
Ma0                            up             up
sw-1#
```

Our environment is up and running! Let's hop into Chapter 1 to do some Ansible stuff:
```
git switch chapter-1
```
https://github.com/emieli/ansible-demo/tree/chapter-1
