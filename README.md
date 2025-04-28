# Chapter 0: Getting started
In this chapter we deploy our lab environment so that we have something we can run ansible commands again. We'll be setting up a Fortigate Firewall and two Arista cEOS L3 switches. Arista CLI is very similar to Cisco syntax. In later chapters we'll be configuring VLANs through Ansible playbooks. The topology looks like this:

```
[ FW-1 ]
 port2
   |
  Et1
[ SW-1 ]
  Et2
   |
  Et1
[ SW-2 ]
```

## Step 1
SSH to our Linux server deployed for this purpose:
```
ssh 10.215.200.25
```
*Contact Emil Boklund if login fails*

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

