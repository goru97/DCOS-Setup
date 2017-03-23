<p align="center">
 <img src="https://raw.githubusercontent.com/goru97/DCOS-Setup/master/logos/rackspace-logo.png" height="150" align=center>
</p>
<p align="center">
 <img src="https://raw.githubusercontent.com/goru97/DCOS-Setup/master/logos/dcos-logo.png" height="150" align=center>
</p>

# DCOS-Setup (Using Rackspace cloud)
Setup DC/OS cluster on Rackspace cloud / on-metal servers.

## System Components

- Bootstrap node:
  - This node is used to bootstrap the DCOS cluster
  
- Master node:
  - Basically, acts as a scheduler for DCOS services / containers.
  
- Public agent:
  - Used to run services like load-balancers, reverse-proxy etc
  
- Private agent: 
  - Used to run all the other services and containers.

## Where to start:

Suppose you want to create a DCOS cluster with following configuraton:
- 1 Master
- 2 Public agents
- 3 Private agents

Provision 7 (1 + 2 + 3 + 1bootstrap) rackspace cloud/on-metal centOS servers with flavors of your choice.

## Steps to install:

On each cluster (bootstrap node isn’t the part of cluster) node, do the following:

- Upgrade CentOS:
  - ``` sudo yum upgrade --assumeyes --tolerant ```
  - ``` sudo yum update --assumeyes ```

- Stop the firewall:
  - ``` sudo systemctl stop firewalld && sudo systemctl disable firewalld ```

- Install data compression utilities:
  - ``` sudo yum install -y tar xz unzip curl ipset ```

- Cluster permissions:
	- ``` sudo sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config && sudo groupadd nogroup ```

### Run the following commands on all the nodes (including bootstrap) to install docker services:

- Enable OverlayFS:
  - ``` 
      sudo tee /etc/modules-load.d/overlay.conf <<-'EOF'
      overlay
      EOF 
      ```
      
- Reboot to reload kernel modules:
  - ``` reboot ```

- Configure yum to use Docker yum repo:
  - ```
    sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
    [dockerrepo]
    name=Docker Repository
    baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
    enabled=1
    gpgcheck=1
    gpgkey=https://yum.dockerproject.org/gpg
    EOF 
    ```
    
- Configure systemd to run the Docker Daemon with OverlayFS
  - ```
    sudo mkdir -p /etc/systemd/system/docker.service.d && sudo tee /etc/systemd/system/docker.service.d/override.conf <<- EOF
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd --storage-driver=overlay
    EOF 
    ```
    
- Install the Docker engine, daemon and service
  - ``` sudo yum install -y docker-engine-1.13.1 ```
  - ``` sudo systemctl start docker ```
  - ``` sudo systemctl enable docker ```


## Configure your cluster:

SSH into your bootstrap node and perform the following steps:

- Create a directory named genconf
  - ``` mkdir -p genconf ```

- Create a configuration file and save as genconf/config.yaml.
  - ```
     ---
     agent_list:
     - 10.184.xxx.xxx
     - 10.184.xxx.xxx
     - 10.184.xxx.xx
     bootstrap_url: http://<bootstrap_ip>:<nginx_port>
     cluster_name: DC/OS
     exhibitor_storage_backend: static
     ip_detect_path: genconf/ip-detect
     master_discovery: static
     master_list:
     - 10.184.xxx.xxx
     process_timeout: 10000
     public_agent_list:
     - 104.239.xxx.xxx
     - 65.61.xxx.xxx
     resolvers:
     - 8.8.8.8
     - 8.8.4.4
     ssh_key_path: genconf/ssh_key
     ssh_port: 22
     ssh_user: root
     telemetry_enabled: false 
     ```
  - Note: 
    - agent_list above has the service net ips of your private dc/os agents.
    - bootstrap_ip is the ip address of the current (bootstrap) node
    - nginx_port is the port at which we are going to bind the nginx container in the last step (I use 8080).
    - master_list is the list of service_net addresses of the master nodes.
    - public_agent_list is the list of the ip addresses of public agents.

- Create a ip-detect script and save as genconf/ip-detect.
  - Below is the ip-detect script for Rackspace cloud CentOS distro:
  -  ```
     #!/usr/bin/env bash
     set -o nounset -o errexit
     export PATH=/usr/sbin:/usr/bin:$PATH
     echo $(ifconfig | grep inet | grep -Eo '10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' 
     ```
     
- Download the DC/OS installer
  - I am using the 1.9-RC2 version, you can use the stable release if you like.
  - ``` wget https://downloads.dcos.io/dcos/EarlyAccess/commit/7f1ce42734aa54053291f403d71e3cb378bd13f3/dcos_generate_config.sh?_ga=1.172237866.840181750.1489818130 ```
    
  - Rename the file to dcos_generate_config.sh or use curl –O to download.

- Run the installer:
  - ``` sudo bash dcos_generate_config.sh ```
  - At this point your directory structure should look like:
  ``` 
  ├── dcos-genconf.<HASH>.tar
  ├── dcos_generate_config.sh
  ├── genconf
  │   ├── config.yaml
  │   ├── ip-detect
  ```

- Run the nginx container:
  - ``` sudo docker run -d -p <your-port>:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx ```
  
SSH into your cluster (master and agent nodes) and perform the following steps:
	
- Make a new directory and navigate to it: 
  - ``` mkdir /tmp/dcos && cd /tmp/dcos ```

- Download DC/OS installer from the NGINX docker container:
  - ``` curl -O http://<bootstrap-ip>:<your_port>/dcos_install.sh ```

- This step is different for different node type:
  - For master nodes:
    - ``` sudo bash dcos_install.sh master ```
    
  - For private agent nodes:
    - ``` sudo bash dcos_install.sh slave ```
  - For public agent nodes:
    - ``` sudo bash dcos_install.sh slave_public ```

## Add users to the cluster

- Make sure you have python3 and related modules (kazoo) installed on one of your master nodes.

- ssh into that node and run the following command:
  - ``` python /opt/mesosphere/bin/dcos_add_user.py <github_primary_email_of_the_user> ```




