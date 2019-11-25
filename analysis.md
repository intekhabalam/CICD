
Objective:

This document details the implementation steps of CICD pipeline using a Ansible Playbook, Infra monitoring, app/service monitoring, Nginx Containerization and ELK (Log monitoring etc).
Purpose:
By following this repository we can able to setup a DevOps CI/CD Pipeline.
git
Jenkins
Maven
Ansible
Docker 
Kubernetes ( need to start) 
Sensu for Infra Monitoring
Consul for service Monitoring
ELK for log monitoring and users visualize data with charts and graphs in Kibana.


Here are the steps to run nginx in a docker container with port 9000.


1) Create a docker container to run a simple
Nginx web server in port 9000.


Created an AWS instance with Ubuntu 16.


Installed Docker on it.


Logged into the docker hub using “docker login” command.


Made a Dockerfile as below: Please ignore the # in that. It is just for understanding. Exposed the port 9000 as per the requirement.


========================
FROM ubuntu:18.04


# Install Nginx.
RUN \
  apt-get update && \
  apt-get install -y nginx && \
  rm -rf /var/lib/apt/lists/* && \
  echo "\ndaemon off;" >> /etc/nginx/nginx.conf && \
  chown -R www-data:www-data /var/lib/nginx


# Defined mountable directories.
VOLUME ["/etc/nginx/sites-enabled", "/etc/nginx/certs", "/etc/nginx/conf.d", "/var/log/nginx", "/var/www/html"]


# Defined working directory.
WORKDIR /etc/nginx


# Defined default command.
CMD ["nginx"]


# Expose ports.
EXPOSE 9000
======================


Docker file is created. Here is the command used to build an image.


sudo docker build -t alam1988/nginx_1.1 .


Images has been created now:


root@ip-172-17-2-6:/opt/deploy/myapp# docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
alam1988/nginx_1.1   latest              ac8047a56d76        29 minutes ago      124MB
ubuntu               18.04               775349758637        2 weeks ago         64.2MB


Command to create a docker container using the created images and expose port 9000.


sudo docker run -d -p 9000:80 --name=alam alam1988/nginx_1.1


Nginx container has been created now and running on port 9000.


==================
root@ip-172-17-2-6:/opt/deploy/myapp# docker ps
CONTAINER ID        IMAGE                COMMAND             CREATED             STATUS              PORTS                            NAMES
e2ac3e46b77d        alam1988/nginx_1.1   "nginx"             29 minutes ago      Up 29 minutes       9000/tcp, 0.0.0.0:9000->80/tcp   alam
==================


Now let's start CICD work.

Git :

credentials: 
intekhabalam87@gmail.com
Qazplm123abc+123

https://github.com/intekhabalam/CICD

AWS:

Created 5 instances in AWS for the below: 

-> Jenkins & Git
-> Ansible
-> Docker
-> Tomcat
-> ELK

For launching servers in AWS environment i used t2 micro instance , default VPC, Amazon Linux AMI. Allowed only the required ports from Security ports.

ELK I have setup in ubuntu 18 with t2 large as it needs a good specification of server.

First we need to set up Jenkins Server, Tomcat, Docker

Jenkins Setup:

We will be using open java for our demo, Get the latest version from http://openjdk.java.net/install/

yum install java-1.8*

#yum -y install java-1.8.0-openjdk

Confirm Java Version and set the java home
java -version
find /usr/lib/jvm/java-1.8* | head -n 3
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-1.el8_0.x86_64

export JAVA_HOME
PATH=$PATH:$JAVA_HOME

 # To set it permanently update your .bash_profile
vi ~/.bash_profile
The output should be something like this,

[root@~]# java -version
openjdk version "1.8.0_151"
OpenJDK Runtime Environment (build 1.8.0_151-b12)
OpenJDK 64-Bit Server VM (build 25.151-b12, mixed mode)

Install Jenkins

We can install jenkins using the rpm or by setting up the repo. We will set up the repo so that we can update it easily in the future.

yum -y install wget
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum -y install jenkins

Start Jenkins
# Start jenkins service
service jenkins start
 
# Setup Jenkins to start at boot,
chkconfig jenkins on
Configure Git pulgin on Jenkins

Installed git packages on jenkins server
 yum install git -y
 
Setup GIT on jenkins console
Installed git plugin without restart
Manage Jenkins > Jenkins Plugins > available > github

Configured git path

Manage Jenkins > Global Tool Configuration > git
Installed Maven in my Jenkins and setup M2_HOME and M2 paths of the user.
Steps to setup Maven on jenkins console

Manage Jenkins > Jenkins Plugins > available > Maven Invoker

Ansible Setup:

An AWS EC2 instance (on Control node)
Installation steps:
on Amazon EC2 instance
Install python and python-pip
yum install python
yum install python-pip
Install ansible using pip check for version
pip install ansible
ansible --version
Created a user called ansadmin (on Control node and Managed host)
useradd ansadmin
passwd ansadmin
4.Below command grant sudo access to ansadmin user. But it is  strongly recommended using "visudo" command if you are aware vi or nano editor. (on Control node and Managed host).
echo "ansadmin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
5.  Log in as a ansadmin user on master and generate ssh key (on Control node)
ssh-keygen
6. Copy keys onto all ansible managed hosts (on Control node)
ssh-copy-id ansadmin@<target-server>
7. Ansible server used to create images and store on docker registry. Hence install docker, start docker services and add ansadmin to the docker group.
yum install docker
 
# start docker services 
service docker start
 
# add user to docker group 
usermod -aG docker ansadmin
8: Create a directory /etc/ansible and create an inventory file called "hosts" add control node and managed hosts IP addresses to it.
Validation test
Run ansible command as ansadmin user it should be successful (Master)
ansible all -m ping



Docker Setup:


Install docker and start docker services
yum install docker -y
docker --version 
 
# start docker services
service docker start
service docker status
Create a user called dockeradmin
useradd dockeradmin
passwd dockeradmin
add a user to docker group to manage docker
usermod -aG docker dockeradmin
Validation test
Create a tomcat docker container by pulling a docker image from the public docker registry
docker run -d --name test-tomcat-server -p 8090:8080 tomcat:latest
 
Dockerfile:

FROM tomcat:latest
COPY ./webapp.war /usr/local/tomcat/webapps

Playbooks:

1) create-simple-devops-image.yml


---
- hosts: all
  become: true


  tasks:

  - name: create a docker file using war file
    command: docker build -t simple-devops-image .
    args:
      chdir: /opt/docker
  
  - name: create a tag to image
    command: docker tag simple-devops-image alam1988/simple-devops-image

  - name: push the image to docker hub
    command: docker push alam1988/simple-devops-image
    ignore_errors: yes

  - name: remove docker images from ansible server 
    command: docker rmi simple-devops-image alam1988/simple-devops-image
    ignore_errors: yes


2)
 
create-simple-devops-project.yml

---
- hosts: all
  become: true

  tasks:
  
  - name: stop current container
    command: docker stop simple-devops-container
    ignore_errors: yes

  - name: remove stopped container 
    command: docker rm simple-devops-container
    ignore_errors: yes

  - name: remove docker images
    command: docker rmi alam1988/simple-devops-image
    ignore_errors: yes

#  - name: create docker images from war file
#    command: docker build -t simple-docker-image .
#    args: 
#      chdir: /opt/docker

  - name: pull docker image from docker hub
    command: docker pull alam1988/simple-devops-image

  - name: create docker container using docker image
    command: docker run -d --name simple-devops-container -p 8080:8080 alam1988/simple-devops-image

Above 2 are my Ansible Playbook. 


Now I have integrated Ansible in Jenkins so when I do push my code, my Jenkins job “Final-CICD-pipeline-using-Ansible” start a build and deploy it automatically in a docker container. I will explain in the call how I have configured the job and what are the playbooks I am running in that.

Here are the steps to integrate Ansible in jenkins.

Install a Plugin called "publish Over SSH".

 Manage Jenkins > Manage Plugins > Available > Publish over SSH

Enable connection between Ansible and Jenkins

Manage Jenkins > Configure System > Publish Over SSH > SSH Servers

SSH SERVER
HOSTNAME : serverip
username:  ansadmin
password: *******

Test the connection "Test Connection"
 
Command to check the syntax of the playbook.
 
ansible-playbook -i hosts create-simple-devops-project.yml --check
 
command to run the playbook.
 
ansible-playbook -i hosts create-simple-devops-project.yml --limit 172.31.31.243


INTRODUCTION TO CONSUL

In a traditional architecture for delivering an application, there is a monolithic app(i.e a single application) that has multiple discrete subcomponents. Even though these subcomponents are independent in managing its own process, these are deployed as a single unit. Suppose if there is any bug in any one of the subcomponents, we have to coordinate it with all the other components and redeploy the whole unit. 

Instead of doing this, we deploy them as a discrete services called a microservices or service oriented architecture. Here, if any component needs to be fixed, it can just be patched and deployed without coordinating with the other components.

SERVICE DISCOVERY

There is a challenge that how do these microservices discover each other. Service discovery can be used to solve this problem, but as usual, there are many different ways to implement it

Client Side Service Discovery

One solution is to have a central service registry where all service instances are registered. Clients would have to implement logic to query for a service they need, eventually validate if the endpoints are still alive and maybe distribute requests to multiple endpoints.

Server Side / Load balancing

All traffic goes through a load balancer which knows all the actual, dynamically changing endpoints and redirects all requests accordingly.

HEALTH CHECKS

Health checks in  Consul can be used to monitor the state of all services within a cluster, but also to automatically remove unhealthy service endpoint registrations from the Consul registry. Co

KEY/VALUE

Configuring a pool of services somewhere in the cloud can be tricky. Keeping it up to date or making a change simultaneously in many services is hard.  Instead of trying to define configuration in many individual pieces(services) distributed throughout our infrastructure, we define a key centrally and configure these pieces dynamically.
ARCHITECTURE:



PREREQUISITES FOR INSTALLATION

3 or 5 Linux Servers.
The ports are to be opened between all these servers. In AWS, these ports are to be added in Security groups and firewall tags added properly to allow communications of the below mentioned ports.
8300 -------- TCP
8301 --------- TCP & UDP
8302--------- TCP & UDP
8400---------TCP
8500---------TCP
8600----------TCP & UDP

SETUP CONSUL CLUSTER 

This document is based on a 3 node cluster. The nodes are as follows. 

ConsulM01
ConsulC02
ConsulC03

The following steps are to be performed on all the three nodes. Except step 7.

    1  cd /usr/local/bin
    2  sudo wget https://releases.hashicorp.com/consul/1.4.2/consul_1.4.2_linux_amd64.zip
    3  sudo unzip consul_1.4.2_linux_amd64.zip
    4  sudo rm -rf consul_1.4.2_linux_amd64.zip
    5  sudo mkdir -p /etc/consul.d/scripts
    6  sudo mkdir /var/consul
    7  consul keygen (Only on master server )
    8  sudo vi /etc/consul.d/config.json



{
"bootstrap_expect": 3,
"client_addr": "0.0.0.0",
"datacenter": "[datacenter ]", 
"data_dir": "/var/consul",
"domain": "[domain name]", /* The default domain name is ‘consul’*/
"enable_script_checks": true,
"dns_config": {
"enable_truncate": true,
"only_passing": true
},
"enable_syslog": true,
"encrypt": "<replace with 7th step value>",
"leave_on_terminate": true,
"log_level": "INFO",
"rejoin_after_leave": true,
"server": true,
"start_join": [
"node-01-IP",
"node-02-IP",
"node-03-IP"
],
"ui": true
}
   
   
   9  sudo vi /etc/systemd/system/consul.service

[Unit]
Description=Consul Startup process
After=network.target
 
[Service]
Type=simple
ExecStart=/bin/bash -c '/usr/local/bin/consul agent -config-dir /etc/consul.d/'
TimeoutStartSec=0
 
[Install]
WantedBy=default.target



/* Start the consul on all the three services.*/
   10  sudo systemctl daemon-reload
   11  sudo systemctl start consul
   12 sudo systemctl enable consul

CONSUL VALIDATE:

This is used to validate the configuration without actually starting the agent.
 
~ consul validate /etc/consul.d

/* Check the consul cluster*/

~ Consul members



It means your cluster is up and running. 

ACCESS CONSUL UI

From Consul version 1.2 UI is an inbuilt conssl component.

Access the econsul web UI using the following syntax in your browser.

http://consul-ip:8500/ui

You can view an UI as shown below.


      
I have already configured sensu in one of my server, so I just added the sensu agent in my client machine to monitor the infra, ie; memory, cpu, disk usage, etc.



I have also configured ELK to see the logs from clinet mahcine to Kibana UI and users visualize data with charts and graphs in Kibana.




