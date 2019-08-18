# ansible-deploy-ec2-with-lamp
###### Ansible playbook to deploy an ec2 instance with fully configured lamp. This can be used to deploy a full WordPress installation or any CMS/Application that works over a LAMP stack.

### Prerequisite:
1. Install Ansible on an ec2 Instance and setup it as Ansible-master
2. Python boto library
3. Create an IAM Role with Policy AmazonEC2FullAccess and attach it to the Ansible master instance.

##### This playbook will create a new security group and deploy an ec2 instance, then install lamp stack on it. 
----
NB: This will only work in deploying a single instance with lamp stack.

---
##### ec2-lamp.yml
```sh
---
 - name: "Deploying ec2 instances"
   hosts: localhost

   vars:
     instance_type: t2.micro
     security_group: ansible-sg
     image: ami-0b898040803850657
     region: us-east-1
     keypair: jwl-pc
     exact_count: 1
     subnet: subnet-5074287e
     hostkey: /home/ec2-user/jwl-pc.pem
     mysql_root_password: mysqlroot123
     mysql_user: wpuser
     mysql_password: wpuser123
     mysql_database: wordpress

   tasks:
     - name: Creating a security group
       ec2_group:
         name: "{{ security_group }}"
         description: Ansible-webserver
         region: "{{ region }}"
         rules:
           - proto: tcp
             from_port: 22
             to_port: 22
             cidr_ip: 0.0.0.0/0
           - proto: tcp
             from_port: 80
             to_port: 80
             cidr_ip: 0.0.0.0/0
           - proto: tcp
             from_port: 443
             to_port: 443
             cidr_ip: 0.0.0.0/0
         rules_egress:
           - proto: all
             cidr_ip: 0.0.0.0/0

     - name: "Launching ec2 Instance"
       ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{keypair}}"
         image: "{{image}}"
         region: "{{region}}"
         group: "{{security_group}}"
         vpc_subnet_id: "{{subnet}}"
         wait: yes
         count_tag:
           Name: Ansible-slave
         instance_tags:
           Name: Ansible-slave
         exact_count: "{{exact_count}}"
       register: ec2_status
       
     - debug:
         var: item.public_ip
       with_items: "{{ ec2_status.instances }}"

     - name: "Adding host to inventory"
       add_host:
        hostname: webserver
        ansible_host: "{{ ec2_status.instances.0.public_ip }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ hostkey }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"


     - name: Waiting for OpenSSH to open"
       wait_for:
         port: 22
         host: "{{ ec2_status.instances.0.public_ip }}"
         timeout: 80
         state: started
         delay: 10

     - name: "Apache - Installing"
       delegate_to: webserver
       become: yes
       yum: name=httpd,php,php-mysql state=present

     - name: "Apache - Creating index.html"
       delegate_to: webserver
       become: yes
       copy:
         content: " <h1><center> Awesome server running on {{ ec2_status.instances.0.public_ip }} </center></h1>"
         dest: /var/www/html/index.html

     - name: "Apache - Creating phpinfo.php"
       delegate_to: webserver
       become: yes
       copy:
         content : "<?php phpinfo(); ?>"
         dest : /var/www/html/phpinfo.php

     - name: "Apache - Restarting/Enabling"
       delegate_to: webserver
       become: yes
       service:
         name: httpd
         state: restarted
         enabled: yes

     - name: 'Mariadb-server Installing'
       delegate_to: webserver
       become: yes
       yum: name=MySQL-python,mariadb-server state=present

     - name: 'Mariadb-server Restarting/Enabling '
       delegate_to: webserver
       become: yes
       service: name=mariadb state=restarted enabled=yes

     - name: 'Mariadb-server Resetting Root Password'
       delegate_to: webserver
       become: yes
       ignore_errors: true
       mysql_user:
         login_user: root
         login_password: ''
         name: root
         password: "{{mysql_root_password}}"
         host_all: true

     - name: 'Mariadb-server Removing Anonymous users'
       delegate_to: webserver
       become: yes
       mysql_user:
         login_user: root
         login_password: "{{mysql_root_password}}"
         name: ''
         state: absent
         host_all: true


     - name: 'Mariadb-server Creating Additional database'
       delegate_to: webserver
       become: yes
       mysql_db:
         login_user: root
         login_password: "{{mysql_root_password}}"
         name: "{{mysql_database}}"
         state: present

     - name: 'Mariadb-server Creating additional user/password'
       delegate_to: webserver
       become: yes
       mysql_user:
         login_user: root
         login_password: "{{mysql_root_password}}"
         name: "{{mysql_user}}"
         password: "{{mysql_password}}"
         state: present
         host: localhost
         priv: "{{mysql_database}}.*:ALL"
```
---
#### Variables
----
```sh
     instance_type: 
     security_group: 
     image: 
     region: 
     keypair: 
     exact_count: 
     subnet: 
     hostkey: 
     mysql_root_password: 
     mysql_user: 
     mysql_password: 
     mysql_database: 

  ```
---

##### Executing the playbook - # ansible-playbook ec2-lamp.yml

