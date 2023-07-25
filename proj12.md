# ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)

### Step 1 – Jenkins job enhancement

### creating a new directory called ansible-config-artifact in the jenkins-ansible server to store all artifacts after each build

`sudo mkdir /home/ubuntu/ansible-config-artifact`

![Alt text](<Images/Screenshot 2023-07-24 175915.png>)

### Changing permissions to this ansible-config-artifact directory, so Jenkins could save files there 

`chmod -R 0777 /home/ubuntu/ansible-config-artifact`

![Alt text](<Images/Screenshot 2023-07-24 180527.png>)

### Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins

![Alt text](<Images/Screenshot 2023-07-24 180947.png>)

### Creating a new Freestyle project save_artifacts to save artifacts project into /home/ubuntu/ansible-config-artifact directory

![Alt text](<Images/Screenshot 2023-07-24 181936.png>)

![Alt text](<Images/Screenshot 2023-07-24 182255.png>)

![Alt text](<Images/Screenshot 2023-07-24 182359.png>)

### creating a Build step and choose Copy artifacts from other project, specify ansible as a source project and /home/ubuntu/ansible-config-artifact as a target directory.

![Alt text](<Images/Screenshot 2023-07-24 183439.png>)

![Alt text](<Images/Screenshot 2023-07-24 182255.png>)

### Step 2 – Refactor Ansible code by importing other playbooks into site.yml

1. Within playbooks folder, create a new file and name it site.yml – This file will now be considered as an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference.

2. Create a new folder in root of the repository and name it static-assignments. The static-assignments folder is where all other children playbooks will be stored

3. Move common.yml file into the newly created static-assignments folder.

4. Inside site.yml file, import common.yml playbook.

  ---
- hosts: all
- import_playbook: ../static-assignments/common.yml`

### The code above uses built in import_playbook Ansible module.

![Alt text](<Images/Screenshot 2023-07-24 205145.png>)

5. Run `ansible-playbook` command against the dev environment

### Since we will need to apply some tasks to your `dev` servers and `wireshark` is already installed – we will go ahead and create another playbook under `static-assignments` and name it `common-del.yml`. In this playbook, configure deletion of `wireshark` utility.

---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes


![Alt text](<Images/Screenshot 2023-07-24 213146.png>)


![Alt text](<Images/Screenshot 2023-07-24 213920.png>)

### update `site.yml` with `- import_playbook: ../static-assignments/common-del.yml` instead of `common.yml` and run it against `dev` servers:    

## CONFIGURE UAT WEBSERVERS WITH A ROLE ‘WEBSERVER

1. Launch 2 fresh EC2 instances using RHEL 8 image, we will use them as our uat servers, so give them names accordingly – Web1-UAT and Web2-UAT

![Alt text](<Images/Screenshot 2023-07-25 202634.png>)

2. To create a role, you must create a directory called roles/, relative to the playbook file or in /etc/ansible/ directory

Use an Ansible utility called ansible-galaxy inside ansible-config-mgt/roles directory (you need to create roles directory upfront)

`mkdir roles`
`cd roles`
`ansible-galaxy init webserver`

Create the directory/files structure manually

![Alt text](<Images/Screenshot 2023-07-25 203532.png>)

![Alt text](<Images/Screenshot 2023-07-25 203839.png>)

3. Update your inventory ansible-config-mgt/inventory/uat.yml file with IP addresses of your 2 UAT Web servers

[uat-webservers]
<Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' 

<Web2-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user'

4. In /etc/ansible/ansible.cfg file uncomment roles_path string and provide a full path to your roles directory roles_path    = /home/ubuntu/ansible-config-mgt/roles, so Ansible could know where to find configured roles.

![Alt text](<Images/Screenshot 2023-07-25 213824.png>)

5. It is time to start adding some logic to the webserver role. Go into tasks directory, and within the main.yml file, start writing configuration tasks to do the following:

Install and configure Apache (httpd service)
Clone Tooling website from GitHub https://github.com/<your-name>/tooling.git.
Ensure the tooling website code is deployed to /var/www/html on each of 2 UAT Web servers.
Make sure httpd service is started
Your main.yml may consist of following tasks:

![Alt text](<Images/Screenshot 2023-07-25 214844.png>)

## REFERENCE WEBSERVER ROLE

### Step 4 – Reference ‘Webserver’ role

Within the static-assignments folder, create a new assignment for uat-webservers uat-webservers.yml. This is where you will reference the role.

![Alt text](<Images/Screenshot 2023-07-25 215916.png>)

Remember that the entry point to our ansible configuration is the site.yml file. Therefore, you need to refer your uat-webservers.yml role inside site.yml.

So, we should have this in `site.yml`

![Alt text](<Images/Screenshot 2023-07-25 220500.png>)

Commit your changes, create a Pull Request and merge them to master branch, make sure webhook triggered two consequent Jenkins jobs, they ran successfully and copied all the files to your Jenkins-Ansible server into /home/ubuntu/ansible-config-mgt/ directory.

Now run the playbook against your uat inventory and see what happens

`sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/uat.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yaml`



