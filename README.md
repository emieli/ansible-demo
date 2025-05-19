*Enter chapter-7 branch*
```
git switch chapter-7
```

# Chapter 7: Ansible Vault
So far in the lab we have been adding credentials as cleartext in our inventory files. This is OK if you don't add these files in your GIT repository. To access the credentials the baddy must have access your current Ansible directory. But it's very likely that you want to commit your inventory to GIT. Not only for version control, but also to easily deploy the same Ansible setup on another server.

By using Ansible Vault we can take our sensitive files (or values) and perform "at-rest" encryption. To read each file (or value) it must first be unlocked with Ansible Vault. I am personally a fan of encrypting specific values instead of the full file, but either method is equally safe. 

Ansible Vault gives us access to the operations explained below. Some of them include opening a text editor and the default option is vi/vim. If you prefer Nano as your editor, run the following commands before you continue with this chapter:
```
(venv) emileli@clab:~/ansible-demo$ echo "export EDITOR=nano" >> ~/.bashrc
(venv) emileli@clab:~/ansible-demo$ bash
emileli@clab:~/ansible-demo$ source venv/bin/activate
(venv) emileli@clab:~/ansible-demo$
```
*Ansible-vault will use Nano as its editor*

## Create & View
We can create a new file with Ansible-vault, encrypt it with a password, edit its contents and finally save to disk. Once saved, the file is encrypted. Let's do an example together. We will create a **test-file**. In this lab I recommend setting the password to **ansible**.  
On the first line we enter the words **test-contents** and then close the file. We then run **cat test-file** to see that the file has been encrypted by ansible-vault. If we instead read the file with **ansible-vault view test-file** and provide the correct password, the contents suddenly become clear to us.
```
(venv) emileli@clab:~/ansible-demo$ ansible-vault create test-file
New Vault password:
Confirm New Vault password:
(venv) emileli@clab:~/ansible-demo$ cat test-file
$ANSIBLE_VAULT;1.1;AES256
63633964663866623839363234366537333634323066633464343365316231393862613439393435
3334366139643565336434343261313832306139326637350a636238623037383164666265326332
64313563383138303061646562346432623032353763346662316162353938393832663563383535
3462616433303862370a666134313266323462616538363562326564396265656230323865373962
3466
(venv) emileli@clab:~/ansible-demo$ ansible-vault view test-file
Vault password:
test-contents
(venv) emileli@clab:~/ansible-demo$
```

## Edit
We can edit an already encrypted file with **ansible-vault edit test-file**. Add a second line containing **more text-contents**, save and close the editor. Use **ansible-vault view test-file** to verify that the file was updated.

## Encrypt
Ansible-vault is also able to encrypt an existing file. As our **hosts.yml** contain sensitive credentials, we should encrypt it. Let's do that together (Set password to **ansible**):
```
(venv) emileli@clab:~/ansible-demo$ cp hosts.yml hosts.yml_backup    # better safe than sorry
(venv) emileli@clab:~/ansible-demo$ ansible-vault encrypt hosts.yml
New Vault password:
Confirm New Vault password:
Encryption successful
(venv) emileli@clab:~/ansible-demo$ cat hosts.yml
$ANSIBLE_VAULT;1.1;AES256
38643439316434346564303862376564636563613336323531393533343232316461353536393130
3863613933373736613736653939363335313166663730370a613538303536336130633738396463
[...]
63306334356131643663346535643737663437643632623839613938626463663762343938666164
633532343439303930313739396363643432
(venv) emileli@clab:~/ansible-demo$ ansible-vault view hosts.yml
Vault password:
SITE01:
  hosts:
    FW-1:
      ansible_host: 172.20.20.2
[...]
```

We can now safely commit our inventory file to GIT as the sensitive credentials are safely encrypted inside the file. However, using GIT as version control is now virtually useless as it can't track the changes inside the file. All GIT sees is what we see when we run the **cat** command. It's also up to you to keep the password in a safe location.

Attempting to run a playbook suddenly became much harder as Ansible is no longer able to read the contents of the inventory file:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml
[WARNING]:  * Failed to parse /home/emileli/ansible-demo/hosts.yml with yaml plugin: Attempting to decrypt but no vault secrets found
```

We need to supply the password used to encrypt **hosts.yml** and run the playbook. We do that with the **--ask-vault-pass** flag:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml --ask-vault-pass
Vault password:

PLAY [Show Arista VLANs]

TASK [Arista: Get vlans]
ok: [SW-1]
ok: [SW-2]
```

The playbook was able to run because Ansible was able to read the inventory file.

## Decrypt
Let's restore **hosts.yml** back to its cleartext format with the following sequence of commands:
```
(venv) emileli@clab:~/ansible-demo$ ansible-vault decrypt hosts.yml
Vault password:
Decryption successful
(venv) emileli@clab:~/ansible-demo$ cat hosts.yml
SITE01:
  hosts:
    FW-1:
      ansible_host: 172.20.20.2
[...]
```

## Encrypt_string
This is my preferred way of encrypting secrets. It strikes a good balance between security and version control as we encrypt specific values in a file instead of encrypting the full file. This allows us to perform GIT version control on our inventory file while also keeping sensitive information inside it. 

Let us encrypt the highly sensitive **ansible_password: admin** value in **hosts.yml**. The process is a bit more cumbersome as we have to generate the string and replace the corresponding line in the file with the generated output:
```
(venv) emileli@clab:~/ansible-demo$ ansible-vault encrypt_string "admin" --name "ansible_password"
New Vault password:
Confirm New Vault password:
Encryption successful
ansible_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34316635373136623930663036333639323863633639323063613432336262353461646333383431
          3932653533363336623034376164386366616130376564380a386433643966373235336566303664
          34646462396661373863316362373035643962303863613061663733653364653435646566633065
          3339633665613031330a633063663138666663616163393465626664636631383633613633636139
          6564
(venv) emileli@clab:~/ansible-demo$ vim hosts.yml
< replace line and save file>
(venv) emileli@clab:~/ansible-demo$ cat hosts.yml
[...]
EOS:
  vars:
    ansible_user: admin
    ansible_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34316635373136623930663036333639323863633639323063613432336262353461646333383431
          3932653533363336623034376164386366616130376564380a386433643966373235336566303664
          34646462396661373863316362373035643962303863613061663733653364653435646566633065
          3339633665613031330a633063663138666663616163393465626664636631383633613633636139
          6564
    ansible_become: true
[...]
```

We have now encrypted a specific value in our inventory rather than the full file. I personally prefer this over fully encrypting each file. 

Let's run the playbook to verify that providing the vault password is still required:
```
(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans] 
fatal: [SW-1]: FAILED! =>
    msg: Attempting to decrypt but no vault secrets found
fatal: [SW-2]: FAILED! =>
    msg: Attempting to decrypt but no vault secrets found

PLAY RECAP 
SW-1                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
SW-2                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0

(venv) emileli@clab:~/ansible-demo$ ansible-playbook playbook_arista_vlans_show.yml --ask-vault-pass
Vault password:

PLAY [Show Arista VLANs] 

TASK [Arista: Get vlans] 
ok: [SW-2]
ok: [SW-1]
```
*--ask-vault-pass is still required, very good*

That's it for this chapter! The next chapter is the last one: https://github.com/emieli/ansible-demo/blob/chapter-8/README.md

# Resources
- https://docs.ansible.com/ansible/latest/cli/ansible-vault.html
- https://www.shellhacks.com/ansible-vault-encrypt-decrypt-string/
- https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data
