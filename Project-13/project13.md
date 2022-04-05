# Project 13 - Ansible Dynamic Assignments (Include) and Community Roles
In this project, Ansible dynamic assignments (include) was utilised to deploy to servers.

Ansible has two modes of operation for reusable content: dynamic and static.

If you use any *include** Task (include_tasks, include_role, etc.), it will be dynamic. 
If you use any *import** Task (import_playbook, import_tasks, etc.), it will be static

- All *import* statements are pre-processed at the time playbooks are parsed.
Meaning, when you execute site.yml playbook, Ansible will process all the playbooks referenced during the time it is parsing the statements. This also means that, during actual execution, if any statement changes, such statements will not be considered. Hence, it is static.

- All *include* statements are processed as they are encountered during the execution of the playbook.
Meaning, after the statements are parsed, any changes to the statements encountered during execution will be used

- Take note that in most cases it is recommended to use static assignments for playbooks, because it is more reliable. With dynamic ones, it is hard to debug playbook problems due to its dynamic nature. However, you can use dynamic assignments for environment specific variables as we will be introducing in this project.

## Step 1 - Introducing Dynamic Assignment Into Our structure
- Create a new folder, name it `dynamic-assignments`. Then inside this folder, create `env-vars.yml file`
``` shell
git checkout -b dynamic-assignments
mkdir dynamic-assignments
touch dynamic-assignments/env-vars.yml
```

Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as servername, ip-address etc., we will need a way to set values to variables per specific environment.

For this reason, we will now create a folder to keep each environment’s variables file. 

- Create a new folder `env-vars`, then for each environment, create new YAML files which we will use to set variables.
``` shell
mkdir env-vars
touch env-vars/{dev,stage,uat,prod}.yml
```

- Now paste the instruction below into the `env-vars.yml` file.
``` yaml
vim env-vars.yml
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - '{{ inventory_file | basename }}'
            - dev.yml
            - stage.yml
            - prod.yml
            - uat.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always
```

- The code was updated to ensure it is a task file and not a play file, i.e it doesn't have a "hosts" option. The code does the following:

	- it loops around the file contents in the env-vars directory created above based on the order set and picks the first found file (using the with_first_found)
	- it first checks the inventory_file that was passed to the ansible deployment and returns its basename(without the file path).
	- for instance, if the inventory file to be used with the ansible deployment command i.e ansible -i inventory_file, is /home/ubuntu/ansible-config-artifact/inventory/uat.yml, uat.yml becomes the basename. This makes the env-vars.yml dynamic in choosing the required inventory file depending on the enviroments.('{{ inventory_file | basename }}')
	- it then checks the content of the file and retrieves the variables set in them (include_vars: "{{ item }}")
	
![update_env_vars_yml](Screenshots/update_env_vars_yml.png)

- Update site.yml file to make use of the dynamic assignment.
``` yaml
---
- import_playbook: ../static-assignments/common.yml   
- name: Include dynamic variables
  hosts: all
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- name: Webserver assignment
  import_playbook: ../static-assignments/uat-webservers.yml
```
	
Note: import_playbook can only be used as top level play and cannot be in a tasks option

The tree structure should look the screenshot below:

![tree_structure](Screenshots/tree_structure.png)

- Push to Github and Merge to the main branch. Disable Github Webhook to stop Jenkins builds

## Step 2 - Connect to Jenkins-Ansible server
As there is no need to use Jenkins build for the project, Git would be installed on the Jenkins Ansible server to easily fetch our code from the Github repo and deploy to the servers.

- Navigate to the ansible artifact directory where the jenkins build are copied into and pull the git directly from github
``` bash
cd ansible-config-artifact
git init
git pull https://github.com/amadinathaniel/ansible-config-mgt.git
git remote add origin https://github.com/amadinathaniel/ansible-config-mgt.git
git commit -m "added roles-feature branch"
```
- Configure the github personal details
``` bash
git config --global user.email "amadinathaniel@gmail.com"
git config --global user.name "amadinathaniel"
```

- Switch to a new branch
``` bash
git branch roles-feature
git switch roles-feature
```

## Step 3 - Utilizing Community Roles - MySQL
- Download existing ansible roles
Here we would install the ansible role for mysql

``` bash
chmod -R 777 roles
cd roles
ansible-galaxy install geerlingguy.mysql
```

- Rename the ansible role
``` bash
mv geerlingguy.mysql/ mysql
```

- Update the mysql > defaults > main.yml file and edit roles configuration to use correct credentials for MySQL required for the tooling website.

![modify_defaults_mainyml](Screenshots/modify_defaults_mainyml.png)

- Upload changes into Github and merge to main branch
``` bash
cd ..
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature
```
- Generate Personal Access Token if required to complete the operation above.

## Step 4 - Utilizing Community Roles - Load Balancer Roles
Here, we will try to create a dynamic assignment such that we would be able to select a load balancer type to deploy to the server i.e nginx or apache:
- Install the Nginx and Apache roles:
``` bash
cd roles

ansible-galaxy install geerlingguy.nginx 
mv geerlingguy.nginx/ nginx


ansible-galaxy install geerlingguy.apache 
mv geerlingguy.apache/ apache
```

- Declare a variable in defaults/main.yml file inside the Nginx and Apache roles, to enable the preferred load balancer role:. 

Name each variables `enable_nginx_lb` and `enable_apache_lb` respectively ans set their values to `false`.
``` bash
echo "enable_nginx_lb: false" >> nginx/defaults/main.yml 
echo "load_balancer_is_required: false" >> nginx/defaults/main.yml 

echo "enable_apache_lb: false" >> apache/defaults/main.yml
echo "load_balancer_is_required: false" >> apache/defaults/main.yml 
```

- Create and update loadbalancers.yml in the static-assignments with the conditions.
Here the required role is only used if the variables are both true.
``` yaml
cd ..
cat << EOT > static-assignments/loadbalancers.yml
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
EOT
```

![create_loadbalancer_yml](Screenshots/create_loadbalancer_yml.png)

- Update the site.yml file to import the loadbalancers.yml file on the condition when load_balancer_is_required is true
``` yaml
cat << EOF >> playbooks/site.yml
- name: Loadbalancers assignment
  import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
EOF
```

- playbooks/site.yml should have the following content:

``` yaml
---
- name: Include dynamic variables
  hosts: all
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- name: Webserver assignment
  import_playbook: ../static-assignments/uat-webservers.yml

- name: Loadbalancers assignment
  import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
```
![update_site_yml](Screenshots/update_site_yml.png)

- Update the inventory/uat.yml with the target server details:

![uatweb_lb_inventory](Screenshots/uatweb_lb_inventory.png)

- To activate load balancer, and enable nginx by setting these in the respective environment’s env-vars file.
``` yaml
cat << EOF > env-vars/uat.yml
enable_nginx_lb: true
load_balancer_is_required: true
EOF
```

## Step 5 - Updates - Set Privileged Escalation for Installed roles
To successfully run the nginx and apache roles in the ansible playbook, privilege escalation is required: become: true was used for plays while ansible_become: yes variable was used for tasks
See [here](https://stackoverflow.com/questions/56558841/include-tasks-does-not-work-with-become-after-upgrade-to-ansible-2-8) for similar issue

- roles > apache > tasks > main.yml

![update_apache_main_yml](Screenshots/update_apache_main_yml.png)

- roles > nginx > tasks > main.yml

![update_nginx_main_yml](Screenshots/update_nginx_main_yml.png)

## Step 6 - complete Deployment
- Run the ansible playbook command:
``` bash
ansible-playbook -i /home/ubuntu/ansible-config-artifact/inventory/uat.yml /home/ubuntu/ansible-config-artifact/playbooks/site.yml
```

![deploy_playbook1](Screenshots/deploy_playbook1.png)

- Observe that the apache roles was skipped but nginx was installed on the Load Balancer plays

![deploy_playbook2](Screenshots/deploy_playbook2.png)

![deploy_playbook3](Screenshots/deploy_playbook3.png)

[Back to top](#)



