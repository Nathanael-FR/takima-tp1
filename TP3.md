# Discover Ansible

For our project, we will use Ansible to install and deploy our application automatically.

Ansible inventories define the hosts and groups of hosts upon which commands, modules, and playbooks can be executed. By default, the inventory is stored in `/etc/ansible/hosts`. For our project, we create a custom inventory file to define our production environment.

```
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ~/ssh_keys/id_rsa
 children:
   prod:
     hosts: nathanael.lahary.takima.cloud
```

To test the connectivity to our host using the inventory, we can use the Ansible ping module:

```
ansible all -i inventories/setup.yml -m ping
```

## Advanced Playbook : Install Docker

```
# docker/tasks/main.yml

# Load environment variables from .env file
- name: Load environment variables from .env file
  community.general.env:
    state: present
    envfile: "{{ playbook_dir }}/.env"

# Install device-mapper-persistent-data
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest
  
# Install lvm2
- name: Install lvm2
  yum:
    name: lvm2
    state: latest

# Add Docker repository
- name: Add Docker repository
  command:
    cmd: "sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo"

# Install Docker
- name: Install Docker
  yum:
    name: docker-ce
    state: present

# Install python3
- name: Install python3
  yum:
    name: python3
    state: present

# Install Docker Python module
- name: Install Docker Python module
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

# Ensure Docker service is running
- name: Ensure Docker service is running
  service:
    name: docker
    state: started
  tags: docker

# Log in to Docker Hub
- name: Log in to Docker Hub
  docker_login:
    username: "{{ lookup('env', 'DOCKER_HUB_USERNAME') }}"
    password: "{{ lookup('env', 'DOCKER_HUB_PASSWORD') }}"
    reauthorize: yes
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

```
# playbook.yml

# Define the playbook to be executed on all hosts
- hosts: all

  # Disable gathering facts to speed up the execution
  gather_facts: false
  
  # Enable privilege escalation (become) to execute tasks with root privileges
  become: true

  # Specify the roles to be executed
  roles:
    # Execute the Docker role
    - docker
```

* The playbook runs on all hosts specified in the Ansible inventory.
* Fact collection is disabled (gather_facts: false) to optimize performance, as facts are not needed in this case.
* Privilege (become: true) are granted to allow execution of commands with superuser privileges.
* The Docker role is included (- docker) in the list of roles to apply. This triggers the execution of the docker/tasks/main.yml job file in the Docker role on each target host.
* In the Docker role, environment variables are loaded from the . env, the required packages are installed, Docker is configured and the Docker service is started.

Note : Here we placed the .env file within the directory where the playbook.yml is located. The file looks like this :

```
HOSTNAME=nathanael.lahary.takima.cloud
SSH_PRIVATE_KEY_PATH=~/ssh_keys/id_rsa
DOCKER_TOKEN= * * *  
DOCKER_USERNAME=nathanaelfr
```

* Finally, a connection to Docker Hub is established using the credentials from the environment variables loaded from the .env file.


## Deploy your App

### Network

```
# network/tasks/main.yml

# Create Docker network named 'app-network'
---
- name: Create Docker network
  docker_network:
    name: app-network
    state: present
  vars:
    ansible_python_interpreter: /usr/bin/python3

```

### Databse

```
# database/tasks/main.yml

# Pull the latest Docker image for the database
---
- name: Pull Docker image
  docker_image:
    name: nathanaelfr/tp-devops-database:latest
    source: pull
  vars:
    ansible_python_interpreter: /usr/bin/python3

# Run Docker container named 'myPostgres' using the pulled image within the 'app-network'
- name: Run Docker container in the Docker network
  docker_container:
    name: myPostgres
    image: nathanaelfr/tp-devops-database:latest
    state: started
    networks:
      - name: app-network
  vars:
    ansible_python_interpreter: /usr/bin/python3

```

### App


```
# app/tasks/main.yml

# Pull the latest Docker image for the application
---
- name: Pull Docker image
  docker_image:
    name: nathanaelfr/tp-devops-simple-api:latest
    source: pull
  vars:
    ansible_python_interpreter: /usr/bin/python3

# Run Docker container named 'myStudentApi' using the pulled image within the 'app-network'
- name: Run Docker container in the Docker network
  docker_container:
    name: myStudentApi
    image: nathanaelfr/tp-devops-simple-api:latest
    state: started
    networks:
      - name: app-network
  vars:
    ansible_python_interpreter: /usr/bin/python3

```


### Proxy

```
# proxy/tasks/main.yml

# Pull the latest Docker image for the proxy server
---
- name: Pull Docker image
  docker_image:
    name: nathanaelfr/tp-devops-httpd:latest
    source: pull
  vars:
    ansible_python_interpreter: /usr/bin/python3

# Run Docker container named 'myHttpdServer' using the pulled image within the 'app-network'
# Expose port 80 on the host and map it to port 80 inside the container
- name: Run Docker container in the Docker network
  docker_container:
    name: myHttpdServer
    image: nathanaelfr/tp-devops-httpd:latest
    state: started
    networks:
      - name: app-network
    ports:
      - "80:80"
  vars:
    ansible_python_interpreter: /usr/bin/python3

```
  
These roles are designed to deploy different parts of our "Dockerised" application to a server managed by Ansible. Here’s how they work together:

* The "network" role creates a Docker network named "app-network" that will be used to connect the different containers of oui application.
* The "database" role downloads the Docker image from our PostgreSQL database and launches a container named "myPostgres" using that image. This container is connected to the "app-network" network.
* The "app" role downloads the Docker image of our API application and launches a container named "myStudentApi" using that image. This container is also connected to the "app-network" network.
The "proxy" role downloads the Docker image from our proxy server and launches a container named "myHttpdServer" using that image. This container is connected to the "app-network" network and expose the 80 port on the remote ansible server to enable access to our app.

## Full CI/CD pipelin

```
name: Deploy with Ansible

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.x'

    - name: Install Ansible
      run: |
        python -m pip install --upgrade pip
        pip install ansible

    - name: Install required collections
      run: |
        ansible-galaxy collection install community.general

    - name: Run Ansible Playbook
      env:
        DOCKER_HUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
        DOCKER_HUB_PASSWORD: ${{ secrets.DOCKER_HUB_PASSWORD }}
        
      run: |
        cd ansible
        ansible-playbook -i inventories/setup.yml playbook.yml
```

We add this workflow `deploy.yml` to import the Docker credentials to the environement variables and deploy the app using Ansible.
