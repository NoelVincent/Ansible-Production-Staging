# Deploying from Git to staging and production environments.
This is a simple Ansible playbook (using Roles) which can be used to deploy application from Git to both staging and production environments.

# Ansible
Ansible is a radically simple IT automation engine that automates cloud provisioning, configuration management, application deployment, intra-service orchestration, and many other IT needs. It is the simplest way to automate IT. Ansible is the only automation language that can be used across entire IT teams from systems and network administrators to developers and managers.
You can learn more about ansible : 
- https://www.ansible.com
- https://www.ansible.com/overview/how-ansible-works

# Intro to playbooks
Ansible Playbooks offer a repeatable, re-usable, simple configuration management and multi-machine deployment system, one that is well suited to deploying complex applications. If you need to execute a task with Ansible more than once, write a playbook and put it under source control. Then you can use the playbook to push out new configuration or confirm the configuration of remote systems.
You can learn more about the same in the [website.](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)

# Installing Ansible
> Here I am using AWS Amazon linux server
```sh
sudo amazon-linux-extras install ansible2 -y
```
Please see the [website](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide) for more information.

# Roles in Ansible
Roles let you automatically load related vars, files, tasks, handlers, and other Ansible artifacts based on a known file structure. After you group your content in roles, you can easily reuse them and share them with other users.
You can learn more about the same in the [website.](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)

# 1. Defining Variables in Roles
> Here, I have created two roles "prod-stg" and "deployment" and defining variables that will be used to run in the ansible-playbook.

Defining variables in Role: "prod-stg", that will be used to fetch details of Production and Staging Environment.
```sh
vim prod-stg/vars/main.yml
```
```sh
---
region: "ap-south-1"
access_key: " "   ===========> IAM user access_key
access_key: " "   ===========> IAM user access_key
```

Defining variables in Role: "deployment", that will be used to Fetch and clone from Github.
```sh
vim prod-stg/vars/main.yml
```
```sh
---
clone_dir: "/tmp/website/"
git_url: "https://github.com/NoelVincent/Ansible-Production-Staging.git"
```
# 2. Creating Playbook

```sh
vim main.yml
```
```
---
- name: "Deploying test application on Staging and Production Environment"
  hosts: localhost
  vars_prompt:
    - name: "env"
      prompt: "Enter the environment -(stg) for staging and (prod) for production"
      private: no

    - name: "key_name"
      prompt: "Enter the SSH key name"
      private: no
      default: "ansible"
  roles:
    - prod-stg

############################### Deploying Code - webserver ###############################

- name: "Deploying Code From Github"
  hosts: webserver
  become: true
  serial: 1
  roles:
    - deployment
```
Here, first according to our choice, whether we are deploying our site in a staging or production environment, the playbook will fetch details of the instances. The task for fetching details of these instances are mentioned in the Role:"prod-stg" in "prod-stg/tasks/main.yml".

And then the website files will be fetched and clone from Github and will be updated in the corresponding staging/production server. The task for the same are mentioned in the Role:"deployment" in "deployment/tasks/main.yml".

### Tasks - Role - prod-stg
```sh
vim prod-stg/tasks/main.yml
```
```sh
---
############## Fetching details of Production and Staging Environment #############

- name: "Production Details"
  ec2_instance_info:
    region: "{{ region }}"
    aws_access_key:  "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    filters:
      instance-state-name: [ "running" ]
      "tag:aws:autoscaling:groupName": "production"
  register: production_info

- name: "Staging Details"
  ec2_instance_info:
    region: "{{ region }}"
    aws_access_key:  "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    filters:
      instance-state-name: [ "running" ]
      "tag:aws:autoscaling:groupName": "staging"
  register: staging_info

######################### Creating Dynamic Inventory for Production and Staging Environment ########################

- name: "Dynamic Inventory for Production"
  when: env == "prod"
  add_host:
    groups: "webserver"
    hostname: "{{ item.public_ip_address }}"
    ansible_host: "{{ item.public_ip_address }}"
    ansible_user: "ec2-user"
    ansible_port: 22
    ansible_private_key_file: "{{key_name}}.pem"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
  with_items:
    - "{{production_info.instances}}"

- name: "Dynamic Inventory for Staging"
  when: env == "stg"
  add_host:
    groups: "webserver"
    hostname: "{{ item.public_ip_address }}"
    ansible_host: "{{ item.public_ip_address }}"
    ansible_user: "ec2-user"
    ansible_port: 22
    ansible_private_key_file: "{{key_name}}.pem"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
  with_items:
    - "{{staging_info.instances}}"
```
### Tasks - Role - deployment
```sh
vim deployment/tasks/main.yml
```
```sh
---
##################### Installing Packages #############################
- name: "Installing httpd,php,git"
  yum:
    name:
      - httpd
      - git
      - php
    state: present

####################### Cloning webserver Repository from Github #########################
- name: "clone repository from Github"
  git:
    repo: "{{git_url}}"
    dest: "{{clone_dir}}"
  register: git_info

###################### OFF-LOAD INSTANCE FROM LB AND WAIT FOR CONNECTION DRAINING ###################
- name: "Offloading instances from Load Balancer"
  when: git_info.changed == true
  service:
    name: httpd
    state: stopped
    
- name: "Connection Draining"
  when: git_info.changed == true
  pause:
    seconds: 5

###################### Copying content and Load Instance to LB #############################
   
- name: "Copying contents to document root"
  when: git_info.changed == true
  copy:
    src: "{{clone_dir}}"
    dest: "/var/www/html/"
    owner: "apache"
    group: "apache"
    remote_src: true 

- name: "Load instance back to loadbalancer"
  when: git_info.changed == true
  service:
    name: httpd
    state: restarted
    enabled: true

- name: "Health check"
  when: git_info.changed == true
  pause:
    seconds: 30

#################################### Printing Deployment status #########################################

- name: "Deployment Status - False"
  when: git_info.changed == false
  debug:
    msg: " Already have the latest version."

- name: "Deployment Status True"
  when: git_info.changed == true
  debug:
    msg: "Deployment Successfull"
```
# Execution
> Make sure the Playbook, the SSH key to access client server in working directory of the Ansible Master server.

- Running a syntax check
```sh
ansible-playbook main.yml --syntax-check
```
- Executing the Playbook
```sh
ansible-playbook main.yml
```
