# CI-CD-Project-Jenkins-Ansible-Docker-01

This project note exemplifies a comprehensive DevOps implementation employing GitHub, Docker, Docker Hub, Git, Jenkins, and Ansible. Delve into the seamless integration of these tools to optimize software delivery, automate workflows, and foster collaboration, resulting in accelerated and superior development outcomes.

# Install and create required server for Docker-host and Ansible Server

We will create two server for Docker-host and Ansible Server and install Docker and Ansible.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/0aeb96a8-92fa-49db-9f56-aa8501a4bb97)

# Create ansadmin user
We have to create ansadmin user on both docker and ansible server to run ansible playbook and then give sudo permission.

- On docker-host
```
[root@dockerhost ec2-user]# cat /etc/passwd
ec2-user:x:1000:1000:EC2 Default User:/home/ec2-user:/bin/bash
dockeradmin:x:1001:1001::/home/dockeradmin:/bin/bash
ansadmin:x:1002:1002::/home/ansadmin:/bin/bash
```
- On Ansible-server
```
ansadmin@ansible-server:~$ sudo cat /etc/passwd
ansadmin:x:1001:1001::/home/ansadmin:/bin/bash
```
# Copy SSH Key to Docker-host from ansible-server
ssh-copy-id 172.31.3.248
```
[root@dockerhost ansadmin]# cat .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDfKXE88+ZMMf52VJsbWTv58xxxxxxxxxxxxxxxx99MxT8KQCh1Mp8= ansadmin@ansible-server

# Install Publish over SSH and add SSH server on Jenkins Server
To send build artifact over SSH to ansible server, the installation of the "Publish over SSH" plug-in on the Jenkins server is required.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/ac6f5f39-8fea-43da-aa8f-c4d2b7481e12)

And add ansible server with username and password on Jenkins Server from Manage Jenkins and Configure setting from System configuration.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/d3eceab5-f4cb-496c-a3f1-919413c6e0f4)

# Prepare Artifact Location, Dockerfile and Ansible Playbook

We will push our application artifact in /opt/docker and we will write dockerfile for our application.

```
ansadmin@ansible-server:/opt/docker$ cat Dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```
And we will add hosts for docker-host in ansible hosts file

```
ansadmin@ansible-server:/opt/docker$ cat /etc/ansible/hosts
```
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups
```
[ansible]
172.31.24.126
[dockerhost]
172.31.3.248
```
And we will check our ansible host connection

```
ansadmin@ansible-server:/opt/docker$ ansible all -a uptime
172.31.24.126 | CHANGED | rc=0 >>
 16:09:18 up 25 min,  3 users,  load average: 0.04, 0.04, 0.07
[WARNING]: Platform linux on host 172.31.3.248 is using the discovered Python interpreter at /usr/bin/python3.9, but future
installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.14/reference_appendices/interpreter_discovery.html for more information.
172.31.3.248 | CHANGED | rc=0 >>
 16:09:18 up 25 min,  2 users,  load average: 0.00, 0.00, 0.00
 ```
 
 And then we will prepare ansible playbook to create docker image, tag and push to docker hub.
 ```
 ansadmin@ansible-server:/opt/docker$ cat regapp.yml
---
- hosts: ansible

  tasks:
  - name: create docker image
    command: docker build -t regapp:latest .
    args:
      chdir: /opt/docker
  - name: create tag to push image to docker hub
    command: docker tag regapp:latest kyinaing/regapp:latest

  - name: push docker image to docker hub
    command: docker push kyinaing/regapp:latest
  ```
    
  And we can verify our playbook as below
    
  ```
    ansadmin@ansible-server:/opt/docker$ ansible-playbook regapp.yml --check

    PLAY [ansible] *************************************************************************************************************

    TASK [Gathering Facts] *****************************************************************************************************
    ok: [172.31.24.126]

    TASK [create docker image] *************************************************************************************************
    skipping: [172.31.24.126]

    TASK [create tag to push image to docker hub] ******************************************************************************
    skipping: [172.31.24.126]

    TASK [push docker image to docker hub] *************************************************************************************
    skipping: [172.31.24.126]

    PLAY RECAP *****************************************************************************************************************
    172.31.24.126              : ok=1    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```
Now we will prepare playbook to deploy docker in docker-host and verify.
```
ansadmin@ansible-server:/opt/docker$ cat deploy-regapp.yml
---
- hosts: dockerhost

  tasks:
  - name: stop existing container
    command: docker stop regapp-server
    ignore_errors: yes

  - name: remove container
    command: docker rm regapp-server
    ignore_errors: yes

  - name: remove images
    command: docker image rm kyinaing/regapp:latest
    ignore_errors: yes

  - name: create container
    command: docker run -d --name regapp-server -p 8082:8080 kyinaing/regapp:latest
  ```
  Verify
  ```
  ansadmin@ansible-server:/opt/docker$ ansible-playbook deploy-regapp.yml --check

PLAY [dockerhost] **********************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************
[WARNING]: Platform linux on host 172.31.3.248 is using the discovered Python interpreter at /usr/bin/python3.9, but future
installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.14/reference_appendices/interpreter_discovery.html for more information.
ok: [172.31.3.248]

TASK [stop existing container] *********************************************************************************************
skipping: [172.31.3.248]

TASK [remove container] ****************************************************************************************************
skipping: [172.31.3.248]

TASK [remove images] *******************************************************************************************************
skipping: [172.31.3.248]

TASK [create container] ****************************************************************************************************
skipping: [172.31.3.248]

PLAY RECAP *****************************************************************************************************************
172.31.3.248               : ok=1    changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
```
# Create Job in Jenkins to push docker image to docker hub and to run docker in docker host

Creat new job in jenkins
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/22ac6473-f86c-42d9-a807-fa4a4ed9ae36)

Source Code Configuration
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/f9d430f1-78a6-4199-bffe-e904c117e8d2)

we can set the branch name of git
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/430609ad-61c4-4a5e-9dce-2789876e3cab)

For continous integration, we can set build triggers with Poll SCM.
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/71d6240a-f54f-47dd-b936-1426bf4fce48)
```
* * * * *
``` 
**This is meaning of "every minute" when you say "* * * * *". Perhaps we can also set "H * * * *" to poll once per hour.**

And check Ignore post-commit hooks
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/c67c84b4-f072-4e9a-9ab6-3abd4d372267)

Build Goals and Options to clean install
![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/59fad53c-30e2-42e3-8e97-4c34d77fb256)

Finnally, we will configure post-build actions.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/a75a50e9-5bcd-42c9-bfe4-ce0275434da8)

For the initial run, we need to manually execute the build now command and it does not require any subsequent commit changes from Git because we configure build trigger.

![image](https://github.com/kyinaing/CI-CD-Project-Jenkins-Ansible-Docker-01/assets/12751896/9ee6c6cf-daab-40b0-8471-67300d67e28b)









