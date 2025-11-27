# **Learn Ansible**



[**Youtube Video Link**](https://youtu.be/y2TSR7p3N0M?si=eEfq041rBCEVmJDg)



###### **First of all Pull the Docker Image for the Latest Ubuntu**



*docker pull ansible/ansible:ubuntu1604*



-----



###### **Then create the Network for Docker Images**



*docker network create ansible-net*



-----



###### **Run the Ansible Controller in Docker**



docker run -it \\

&nbsp; --name ansible-controller \\

&nbsp; --hostname ansible-controller \\

&nbsp; --network ansible-net \\

&nbsp; ansible/ansible:ubuntu1604 bash



-----



###### **Run two Ubuntu 16.04 managed nodes**



*docker run -d \\*

  *--name node1 \\*

  *--hostname node1 \\*

  *--network ansible-net \\*

  *ubuntu:16.04 sleep infinity*



*docker run -d \\*

  *--name node2 \\*

  *--hostname node2 \\*

  *--network ansible-net \\*

  *ubuntu:16.04 sleep infinity*



-----



###### **Install SSH inside node1 and node2**



*docker exec -it node1 bash*

*apt-get update*

*apt-get install -y openssh-server*

*mkdir -p /var/run/sshd*

*service ssh start*

*passwd root   # set password = root*

*exit*



*docker exec -it node2 bash*

*apt-get update*

*apt-get install -y openssh-server*

*mkdir -p /var/run/sshd*

*service ssh start*

*passwd root*

*exit*



-----



###### **Inside controller, generate SSH key**



*ssh-keygen -t ed25519 -C "ansible"*



-----



###### **BEFORE COPY: confirm container DNS works**



*ping -c 1 node1*

*ping -c 1 node2*



If ICMP disabled, use:



*getent hosts node1*



-----



###### **Copy SSH keys to managed nodes**



*ssh-copy-id root@node1*

*ssh-copy-id root@node2*



-----



###### **Test passwordless SSH**



*ssh root@node1*

*ssh root@node2*



If it asks password → you failed the key copy step.

Fix it.



-----



###### **Create Ansible inventory**



*echo "\[nodes]*

*node1*

*node2" > inventory.ini*



-----



###### **Test Ansible**



*ansible -i inventory.ini nodes -m ping -u root*



You should get:



node1 | SUCCESS => ...

node2 | SUCCESS => ...



-----



###### **Where to put your playbooks**



*/root/playbooks/*

    *inventory.ini*

    *first\_playbook.yml*



-----



###### **Create your first playbook**



create first\_playbook.yml



---

*- name: Test Playbook on my nodes*

  *hosts: nodes        # the group from inventory.ini*

  *become: yes         # run tasks as root*

  *tasks:*

    *- name: Update apt packages*

      *apt:*

        *update\_cache: yes*



    *- name: Install nginx*

      *apt:*

        *name: nginx*

        *state: present*



-----



###### **Check Syntax also**



*ansible-playbook --syntax-check first\_playbook.yml*



-----



###### **Run your playbook**



*ansible-playbook -i inventory.ini first\_playbook.yml -u root*



-----



###### **Verify**



*ssh root@node1*

*systemctl status nginx*



-----



###### **What is Inventory?**



Inventory is just a list of machines that Ansible knows about and can manage over SSH.

It tells Ansible:



&nbsp;	Which nodes exist

&nbsp;	How to connect to them (username, port, key)

&nbsp;	How to group them for running tasks



-----



###### **Inventory file basics**



By default, it’s a plain text file, usually called inventory.ini or just hosts.



Example for your lab:



\[nodes]

node1

node2





\[nodes] → a group name (you can run playbooks against this group).



node1 and node2 → hostnames or IPs of your managed nodes (these must resolve from the controller).



-----



###### **Adding SSH options to inventory**



If you’re not using default root/passwordless SSH, you can add parameters:



\[nodes]

node1 ansible\_user=root ansible\_ssh\_pass=root ansible\_port=22

node2 ansible\_user=root ansible\_ssh\_pass=root ansible\_port=22



Better practice: use SSH keys instead of passwords, then you don’t need ansible\_ssh\_pass.



-----



###### **Multiple groups**



You can define multiple groups:



*\[webservers]*

*node1*

*node2*



*\[dbservers]*

*db1*

*db2*



*\[all:children]*

*webservers*

*dbservers*



Run playbooks on a specific group:



*ansible -i inventory.ini webservers -m ping*



Or on all nodes:



*ansible -i inventory.ini all -m ping*



-----



###### **Where to put it**



In your Ansible project folder (e.g., /root/playbooks/inventory.ini)



Reference it when running playbooks:



*ansible-playbook -i inventory.ini first\_playbook.yml*

















