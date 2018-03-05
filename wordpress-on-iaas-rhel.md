# POC Scenario: Deploying Wordpress on Azure IaaS VMs (Red Hat Enterprise Linux)

## Table of Contents
* [Abstract](#abstract)
* [Learning objectives](#learning-objectives)
* [Prerequisites](#prerequisites)
* [Linux Distribution](#linux-distribution)
* [Estimated time to complete this module](#estimated-time-to-complete-this-module)
* [Azure CLI](#azure-cli)
* [PoC Setup](#poc-setup)
* [Virtual Machine Creation](#virtual-machine-creation)
* [Connect to the Virtual Machine](#connect-to-the-virtual-machine)
* [Install MariaDB and Galera Cluster](#install-mariadb-and-galera-cluster)
* [Create Internal Load Balancer for Database](#create-internal-load-balancer-for-database)
* [Create External Load Balancer for Web Servers](#create-external-load-balancer-for-web-servers)
* [Create shared storage for Web Content](#create-shared-storage-for-web-content)
* [Reconfigure Apache](#reconfigure-apache)
* [Add Rule to NSGs](#add-rule-to-nsgs)
* [Prepare Wordpress Installation](#prepare-wordpress-installation)
* [Install Wordpress](#install-wordpress)
* [Testing](#testing)


# Abstract

During this module, you will learn about bringing together all the infrastructure components to build a Wordpress Website running on Linux and making it scalable, highly available and secure.

![Screenshot](media/website-on-iaas-http-linux/wordpressdiagram.png)

* Two DB servers will host MariaDB with Galera cluster for replication. In a production scenario a minimum of 3 nodes is required to avoid split-brain scenarios.
* An Azure Internal Load Balancer will distribute the traffic to the DB servers.
* Two Web servers will host Apache.
* The two web servers will connect to one Azure Storage Account via File Services. This storage will mantain the web content consistent across the web server farm.
* Wordpress will be installed on both Web Servers.
* An Azure External Load Balancer will distribute the traffice to the Web servers.

> Note: This document describes the steps for a proof of concept. Additional steps may be required for a production environment

# Learning objectives
After completing the exercises in this module, you will be able to:
* Create Linux Virtual Machine
* Create an Availability Set
* Create and configure a Load Balancer
* Configure a Highly Available MariaDB cluster with Azure Load Balancer
* Configure a Highly Available Wordpress site

# Prerequisites 
* Access to a Azure Subscription
* Access to an authenticated Azure CLI console

# Linux Distribution
* This PoC release is based on the RHEL OS. Minor changes will be necessary for other distributions.

# Estimated time to complete this module
2 hours

# Azure CLI
 * Open an Azure CLI
 * [Azure CLI] Authenticate
 ```bash
 az login
 ```
 * [Azure CLI] Select the correct subscription
 ```bash
 az account set -s <subscriptionid>
 ```


# PoC Setup
 * [Azure CLI] Create a new Resource Group for this PoC
 ```bash
 az group create -n "FastTrackWordpressPoC" -l "West US"
 ```
 * [Azure CLI] Create a new Virtual Network with one subnet
 ```bash
 az network vnet create --name ftpoc-vnet --resource-group FastTrackWordpressPoC --address-prefix 192.168.0.0/16 --subnet-name frontend --subnet-prefix 192.168.1.0/24
 ```
 * [Azure CLI] Create another subnet
 ```bash
 az network vnet subnet create --resource-group FastTrackWordpressPoC --vnet-name ftpoc-vnet --name backend --address-prefix 192.168.2.0/24
 ```


# Virtual Machine Creation
  * Follow the steps bellow to create the 4 servers for this PoC. Do not use the passwords bellow. If possible, use SSH keys for authentication.
  * [Azure CLI] Create an availability set for the database servers
  ```bash
  az vm availability-set create -n ftpoc-database-as -g FastTrackWordpressPoC 
  ```

  * [Azure CLI] Create the first database VM
  ```bash
  az vm create -n ftpoc-database01 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet backend --availability-set ftpoc-database-as --private-ip-address 192.168.2.11 --size Standard_DS1_V2 --os-disk-name ftpoc-database01-disk01 --no-wait
  ```

  * [Azure CLI] Create the second database VM
  ```bash
  az vm create -n ftpoc-database02 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet backend --availability-set ftpoc-database-as --private-ip-address 192.168.2.12 --size Standard_DS1_V2 --os-disk-name ftpoc-database02-disk01 --no-wait
  ```
  * [Azure CLI] Create an availability set for the web servers
  ```bash
  az vm availability-set create -n ftpoc-web-as -g FastTrackWordpressPoC 
  ```

   * [Azure CLI] Create the first WebServer VM
  ```bash
  az vm create -n ftpoc-web01 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet frontend --availability-set ftpoc-web-as --private-ip-address 192.168.1.11 --size Standard_DS1_V2 --os-disk-name ftpoc-web01-disk01 --no-wait
  ```

  * [Azure CLI] Create the second WebServer VM
  ```bash
  az vm create -n ftpoc-web02 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet frontend --availability-set ftpoc-web-as --private-ip-address 192.168.1.12 --size Standard_DS1_V2 --os-disk-name ftpoc-web02-disk01 --no-wait
  ```

# Connect to The Virtual Machine

* Connect via ssh to all the VMs:
```bash
ssh azureadmin@<public ip address>
```
 ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-6.png)

# Install MariaDB and Galera Cluster
From the SSH terminal, execute the following instructions on both DB servers.

  * [SSH DB01,DB02] Install MariaDB and Galera Cluster
  ```bash
  sudo yum install -y rh-mariadb101-mariadb-server-galera mariadb
  ```

 * [SSH DB01,DB02] Open Galera configuration file
  ```bash
  sudo nano /etc/opt/rh/rh-mariadb101/my.cnf.d/galera.cnf
  ```

 * [SSH DB01,DB02] Change wsrep_cluster_address with the Database Server IPs. Remove the "#" from the beginning of the line. 
  ```bash
  wsrep_cluster_address="gcomm://192.168.2.11,192.168.2.12"
  ```
 * [SSH DB01,DB02] CTRL-X to Save file and CTRL-Q to Quit.

 * [SSH DB01,DB02] Create Firewall exceptions:
  ```bash
  sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=4567/tcp --permanent  
  sudo firewall-cmd --zone=public --add-port=4568/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=4444/tcp --permanent
  sudo firewall-cmd --reload
  ```   

 * [SSH DB01] Bootstrap MariaDb service (execute only on server 1)
  ```bash
  sudo scl enable rh-mariadb101 galera_new_cluster
  ```
 * [SSH DB01,DB02] Enable MariaDB service (execute on both servers)
  ```bash
  sudo systemctl enable rh-mariadb101-mariadb.service
  ```  

 * [SSH DB01,DB02] Start MariaDB service (execute on both servers)
  ```bash
  sudo systemctl start rh-mariadb101-mariadb.service
  ```  

 * [SSH DB01] Open mysql locally (first server only)
 ```bash
 mysql --user root
 ```
* [SSH DB01] Create a new database (first server only)
```sql
CREATE DATABASE ftdemo;
```

* [SSH DB01] Create a new user for remote connection and grant privileges (first server only)
```sql
CREATE USER 'ftdemodbuser'@'%' IDENTIFIED BY '<New Password>';
GRANT ALL PRIVILEGES ON *.* TO 'ftdemodbuser'@'%' WITH GRANT OPTION;
```

# Create Internal Load Balancer for Database
  * [Azure CLI] From the Azure CLI Console, create the Load Balancer for Databases
  ```bash
   az network lb create --name ftppoc-db-lb -g FastTrackWordpressPoC --vnet-name ftpoc-vnet --subnet backend --private-ip-address 192.168.2.10 --backend-pool-name ftpoc-dbservers
  ```
  
  * [Azure CLI] Add the Health Probe to the Load Balancer
  ```bash
  az network lb probe create -g FastTrackWordpressPoC --lb-name ftppoc-db-lb -n ftpoc-mysqlprobe --port 3306 --protocol tcp
  ```

  * [Azure CLI] Add Load Balancing Rule
  ```bash
  az network lb rule create -g FastTrackWordpressPoC --lb-name ftppoc-db-lb -n ftppoc-mysqlrule --protocol Tcp --frontend-port 3306 --backend-port 3306 --probe-name ftpoc-mysqlprobe --backend-pool-name ftpoc-dbservers
  ```

  * [Azure CLI] Add the First Database Server NIC to the Load Balancer
  ```bash
  az network nic ip-config update -g FastTrackWordpressPoC --name ipconfigftpoc-database01 --nic-name ftpoc-database01VMNic --lb-name ftppoc-db-lb --lb-address-pools ftpoc-dbservers
  ```

  * [Azure CLI] Add the Second Database Server NIC to the Load Balancer
  ```bash
  az network nic ip-config update -g FastTrackWordpressPoC --name ipconfigftpoc-database02 --nic-name ftpoc-database02VMNic --lb-name ftppoc-db-lb --lb-address-pools ftpoc-dbservers
  ```
  
  # Create External Load Balancer for Web Servers

  * [Azure CLI] Create a public IP for the website. Note that the DNS should be unique.
  ```bash
  az network public-ip create --name ftpoc-web-pip -g FastTrackWordpressPoC --dns-name ftpocweb --allocation-method Dynamic
  ```
 
  * [Azure CLI] Create the external load balancer
  ```bash
  az network lb create --name ftppoc-web-lb -g FastTrackWordpressPoC --backend-pool-name ftpoc-webservers --public-ip-address ftpoc-web-pip
  ```

  * [Azure CLI] Add the Health Probe to the Load Balancer
  ```bash
  az network lb probe create -g FastTrackWordpressPoC --lb-name ftppoc-web-lb -n ftpoc-webprobe --port 80 --protocol tcp
  ```

  * [Azure CLI] Add Load Balancing Rule
  ```bash
  az network lb rule create -g FastTrackWordpressPoC --lb-name ftppoc-web-lb -n ftppoc-httprule --protocol Tcp --frontend-port 80 --backend-port 80 --probe-name ftpoc-webprobe --backend-pool-name ftpoc-webservers
  ```

  * [Azure CLI] Add the First Web Server NIC to the Load Balancer
  ```bash
  az network nic ip-config update -g FastTrackWordpressPoC --name ipconfigftpoc-web01 --nic-name ftpoc-web01VMNic --lb-name ftppoc-web-lb --lb-address-pools ftpoc-webservers
  ```

  * [Azure CLI] Add the Second Web Server NIC to the Load Balancer
  ```bash
  az network nic ip-config update -g FastTrackWordpressPoC --name ipconfigftpoc-web02 --nic-name ftpoc-web02VMNic --lb-name ftppoc-web-lb --lb-address-pools ftpoc-webservers
  ``` 


  # Create shared storage for Web Content
  * [Azure CLI] Create storage account and file share (Azure CLI). The storage account name should be unique.
  ```bash
  az storage account create --sku Standard_LRS -g FastTrackWordpressPoC --name ftpoccontentstorage01
  az storage share create --name webcontent --account-name ftpoccontentstorage01
  ```

  * [SSH WEB01,WEB02] On the web servers, install the utilities to mount SMB:
  ```bash
  sudo yum install cifs-utils -y
  ```

  * [SSH WEB01,WEB02] Create a folder to be a mountpoint
  ```bash
  sudo mkdir /var/webcontent
  ```

  * [Azure Portal] In the Azure Portal, open the storage account -> Overview -> Files. Select the created fileshare "webcontent", click on "Connect" and copy the linux mount command
  * [SSH WEB01,WEB02] Replace the mountpoint on the copied command and execute the command on the webservers:
  ```bash
  sudo mount -t cifs //<storage-account-name>.file.core.windows.net/webcontent /var/webcontent -o vers=3.0,username=<storage-account-name>,password=<storage-account-key>,dir_mode=0777,file_mode=0777,sec=ntlmssp
  ```

  * [SSH WEB01,WEB02] Make mount persistent by executing this command to add a line to /etc/fstab. Replace the storage account name, share, mountpoint and key by the corrent values.
  ```bash
  sudo bash -c 'echo "//<storage-account-name>.file.core.windows.net/<share-name> /var/webcontent cifs nofail,vers=3.0,username=<storage-account-name>,password=<storage-account-key>,dir_mode=0777,file_mode=0777,serverino" >> /etc/fstab'
  ```


  # Reconfigure Apache
  Apache should be already installed on RHEL servers, the following steps will make the necessary adjusments for this PoC.
  * Execute the following steps on both Web Servers
  * [SSH WEB01,WEB02] Install additional packages for apache
  ```bash
  sudo yum install -y php php-common php-mysql php-gd php-xml php-mbstring php-mcrypt -y
  ```
  * [SSH WEB01,WEB02] Enable Apache service
  ```bash
  sudo systemctl enable httpd.service
  ```


* [SSH WEB01,WEB02] Confire SELinux to allow the Web Server to access files on CIFS and to access database
```bash
sudo setsebool -P httpd_use_cifs 1
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1

```

* [SSH WEB01,WEB02] Create a new apache configuration file
```bash
sudo nano /etc/httpd/conf.d/ftdemo.conf
```

* [SSH WEB01,WEB02] Include de following configuration on the new file. Replace the DNS by the one configured in the public IP associated with the External Load Balancer.
```
<VirtualHost *:80>
    ServerName <DNS Name>.westus.cloudapp.azure.com
    DocumentRoot /var/webcontent
</VirtualHost>

<Directory /var/webcontent>
    AllowOverride None
    Require all granted
</Directory>
```
* [SSH WEB01,WEB02] CTRL+O to Save. CTRL+Q to Quit.

* [SSH WEB01,WEB02] Restart apache
```bash
sudo systemctl restart httpd
```

* [SSH WEB01,WEB02] Open Firewall ports
```
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --reload
```
# Add Rule to NSGs
* In the Azure Portal, Open the NSGs associated with the Web Servers (by default its 1 per server)
* Go to "Inbound Security Rules". and add a new rule:
  * Source: any
  * Source port ranges: 
  * Destination: any
  * Destination port ranges: 80
  * Protocol: TCP
  * Action: allow
  * Name: ftpoc-allowhttp

# Prepare Wordpress Installation 

* [SSH WEB01] Execute these steps only on ***Web Server 1***
* [SSH WEB01] Download the latest version of Wordpress and copy contents to the final location
```bash
cd /tmp
wget http://wordpress.org/latest.tar.gz
tar xzf latest.tar.gz
sudo mv wordpress/* /var/webcontent
```

* [SSH WEB01] Open the wordpress configuration file
```
cd /var/webcontent
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

* [SSH WEB01] Change the mysql settings. 
```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'ftdemo');
/** MySQL database username */
define('DB_USER', 'ftdemodbuser');
/** MySQL database password */
define('DB_PASSWORD', '<Password>');
/** MySQL hostname */
define('DB_HOST', '192.168.2.10');
```

# Install wordpress
  * [Browser] Browse to the load balancer public IP dns: **http://(prefix).westus.cloudapp.azure.com/**
  * [Browser] You will see the WordPress Welcome Page
  * [Browser] Fill out the form and click ***Install Wordpress***

   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-16.png)

   * When the installation is completed a success page will appear
   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-23.png)

   * Click ***Log in**, enter your credentials and ensure you can access the Wordpress backoffice
   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-24.png)


# Testing
 * [Browser] Browse to the load balancer public IP dns **http://(prefix).westus.cloudapp.azure.com/**
 ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-25.png)

