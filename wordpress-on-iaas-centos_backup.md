# POC Scenario 2: Deploying Wordpress on Azure IaaS VMs (Red Hat Enterprise Linux) - HTTP

## Table of Contents
* [Abstract](#abstract)
* [Learning objectives](#learning-objectives)
* [Prerequisites](#prerequisites)
* [Estimated time to complete this module](#estimated-time-to-complete-this-module)
* [Customize your Azure Portal](#customize-your-azure-portal)
* [Virtual Machine Creation](#virtual-machine-creation)
* [Connect to the Virtual Machine](#connect-to-the-virtual-machine)
* [Install MariaDB and Galera Cluster](#Install-MariaDB-and-Galera-Cluster)
* [Load Balancer Creation](#load-balancer-creation)
* [Add the VMs to Load Balancer](#add-the-vms-to-load-balancer)
* [Create the load balancing rule for MYSQL](#create-the-load-balancing-rule-for-mysql)
* [Add data disk to Web Servers](#add-data-disk-to-web-servers)
* [Configure Gluster Storage Replication](#configure-gluster-storage-replication)
* [Reconfigure Apache](#reconfigure-apache)
* [Prepare Wordpress Installation](#prepare-wordpress-installation)
* [Install Wordpress](#install-wordpress)
* [Testing](#testing)


# Abstract

During this module, you will learn about bringing together all the infrastructure components to build a Wordpress Website running on Linux and making it scalable, highly available and secure.

![Screenshot](media/website-on-iaas-http-linux/wordpressdiagram-1.png)

* Two DB servers will host MariaDB with Galera cluster for replication. In a production scenario a minimum of 3 nodes is required to avoid split-brain scenarios.
* An Azure Internal Load Balancer will distribute the traffic to the DB servers
* Two Web servers will host Apache. In a production scenario a minimum of 3 nodes is required to avoid split-brain scenarios.
* A data disk will be added to each Web Server
* Gluster storage will be configured on the Web Servers and will replicate the Web Server file content
* Wordpress will be installed on both Web Servers
* An Azure External Load Balancer will distribute the traffice to the Web servers

> Note: This document describes the steps for a proof of concept. Additional steps may be required for a production environment

# Learning objectives
After completing the exercises in this module, you will be able to:
* Create Linux Virtual Machine
* Create an Availability Set
* Create and configure a Load Balancer
* Configure a Highly Available MariaDB cluster with Azure Load Balancer
* Configure a Highly Available Wordpress site with storage replication (gluster)
* Adding a Managed Disk to an existing VM and initializing the disk in Linux

# Prerequisites 
* Access to a Azure Subscription
* Access to an authenticated Azure CLI console

# Linux Distribution
* This PoC release is based on the RHEL OS. Minor changes will be necessary for other distributions.

# Estimated time to complete this module
1.5 hour

# Azure CLI
 * Open an Azure CLI
 * Authenticate
 ```bash
 az login
 ```
 * Select the correct subscription
 ```bash
 az account set <subscriptionid>
 ```


# PoC Setup
 * Create a new Resource Group for this PoC
 ```bash
 az group create -n "FastTrackWordpressPoC" -l "West US"
 ```
 * Create a new Virtual Network with one subnet
 ```bash
 az network vnet create --name ftpoc-vnet --resource-group FastTrackWordpressPoC --address-prefix 192.168.0.0/16 --subnet-name frontend --subnet-prefix 192.168.1.0/24
 ```
 * Create another subnet
 ```bash
 az network vnet subnet create --resource-group FastTrackWordpressPoC --vnet-name ftpoc-vnet --name backend --address-prefix 192.168.2.0/24
 ```


# Virtual Machine Creation
  * Follow the steps bellow to create the 4 servers for this PoC. Do not use the passwords bellow. If possible, use SSH keys for authentication.
  * Create an availability set for the database servers
  ```bash
  az vm availability-set create -n ftpoc-database-as -g FastTrackWordpressPoC 
  ```

  * Create the first database VM
  ```bash
  az vm create -n ftpoc-database01 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet backend --availability-set ftpoc-database-as --private-ip-address 192.168.2.11 --size Standard_DS1_V2 --os-disk-name ftpoc-database01-disk01
  ```

  * Create the second database VM
  ```bash
  az vm create -n ftpoc-database02 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet backend --availability-set ftpoc-database-as --private-ip-address 192.168.2.12 --size Standard_DS1_V2 --os-disk-name ftpoc-database02-disk01
  ```
  * Create an availability set for the web servers
  ```bash
  az vm availability-set create -n ftpoc-web-as -g FastTrackWordpressPoC 
  ```

   * Create the first WebServer VM
  ```bash
  az vm create -n ftpoc-web01 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet frontend --availability-set ftpoc-web-as --private-ip-address 192.168.1.11 --size Standard_DS1_V2 --os-disk-name ftpoc-web01-disk01
  ```

  * Create the second WebServer VM
  ```bash
  az vm create -n ftpoc-web02 -g FastTrackWordpressPoC --image RHEL --admin-username azureadmin --admin-password r3allybadp@ssword --vnet-name ftpoc-vnet --subnet frontend --availability-set ftpoc-web-as --private-ip-address 192.168.1.12 --size Standard_DS1_V2 --os-disk-name ftpoc-web02-disk01
  ```

# Connect to The Virtual Machine

* Connect via ssh to all the VMs:
```bash
ssh azureadmin@<public ip address>
```
 ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-6.png)

# Install MariaDB and Galera Cluster
From the SSH terminal, execute the following instructions on both DB servers.

  * Install MariaDB and Galera Cluster
  ```bash
  sudo yum install -y rh-mariadb101-mariadb-server-galera mariadb
  ```

 * Open Galera configuration file
  ```bash
  sudo nano /etc/opt/rh/rh-mariadb101/my.cnf.d/galera.cnf
  ```

 * Change wsrep_cluster_address with the Database Server IPs. Remove the "#" from the beginning of the line. 
  ```bash
  wsrep_cluster_address="gcomm://192.168.2.11,192.168.2.12"
  ```
 * CTRL-X to Save file and CTRL-Q to Quit.

 * Create Firewall exceptions:
  ```bash
  sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=4567/tcp --permanent  
  sudo firewall-cmd --zone=public --add-port=4568/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=4444/tcp --permanent
  sudo firewall-cmd --reload
  ```   

 * Bootstrap MariaDb service (execute only on server 1)
  ```bash
  sudo scl enable rh-mariadb101 galera_new_cluster
  ```
 * Enable MariaDB service (execute on both servers)
  ```bash
  sudo systemctl enable rh-mariadb101-mariadb.service
  ```  

 * Start MariaDB service (execute on both servers)
  ```bash
  sudo systemctl start rh-mariadb101-mariadb.service
  ```  

 * Open mysql locally (first server only)
 ```bash
 mysql
 ```
* Create a new database (first server only)
```sql
CREATE DATABASE ftdemo;
```

* Create a new user for remote connection and grant privileges (first server only)
```sql
CREATE USER 'ftdemodbuser'@'%' IDENTIFIED BY '<New Password>';
GRANT ALL PRIVILEGES ON *.* TO 'ftdemodbuser'@'%' WITH GRANT OPTION;
```

# Load Balancer Creation
  * From the Azure CLI Console, create the Load Balancer for Databases
  ```bash
   az network lb create --name ftppoc-db-lb -g FastTrackWordpressPoC --vnet-name ftpoc-vnet --subnet backend --private-ip-address 192.168.2.10 --backend-pool-name ftpoc-dbservers
  ```
  
  * Add the Health Probe to the Load Balancer
  ```bash
  az network lb probe create -g FastTrackWordpressPoC --lb-name ftppoc-db-lb -n ftpoc-mysqlprobe --port 3306 --protocol tcp
  ```

* Add Load Balancing Rule
```bash
az network lb rule create -g FastTrackWordpressPoC --lb-name ftppoc-db-lb -n ftppoc-mysqlrule --protocol Tcp --frontend-port 3306 --backend-port 3306 --probe-name ftpoc-mysqlprobe --backend-pool-name ftpoc-dbservers
```


  # Add the VMs to Load Balancer
  * Open the Azure Portal, go to the PoC Resource Group and open the Load Balancer object
  * Under **Settings** select **Backend pools**, click **Add**.
  * Enter name **(prefix)-db-pool**.
  * For **Associated to**, select **Availability set**.
  * For the **Availability set**, select **(prefix)-db-as**.
  * Click **Add a target network IP configuration** to add the first web server and its IP address.

   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-15.png)

  * **Repeat** the step above to also add the IP configuration for the second web server.
  * Click **OK**.

# Create the load balancing rule for MYSQL
  * Under **Settings** select **Load balancing rules**, click **Add**.
  * Enter name **(prefix)-db-lbr**.
    *  Protocol: **TCP**
    *  Port: **3306**
    *  Backend port: 3306
    *  Backend pool: **(prefix)-db-pool(2VMs)**
    *  Probe: **(prefix)-db-prob(HTTP:3306)**
    *  Session Persistence: **None**
    *  Idle timeout (min):**4**
    *  Floating IP (direct server return): **Disabled**
    *  Click **Ok**

   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-22.png)


 # Add data disk to Web Servers
  * Open the Azure Portal
  * Select the First Web Server
  * Select Disks
  * Select "+Add Data Disks"
  * Create a new Managed Disk with 32 Gb with Standard tier

  ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-17.png)

  * Connect to the server via SSH
  * Initialize the data disk using the following procedure: [Initialize Data Disk](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/classic/attach-disk#initialize-a-new-data-disk-in-linux)

  * You can mount the new disk in /datadrive
  * Repeat the steps in the second server

# Configure Gluster Storage Replication

* Execute these steps on both WEB servers
* Add Gluster software repository by creating a new repo file
```bash
nano /etc/yum.repos.d/Gluster.repo
```

* Add the following configuration to the new file
```
[gluster38]
name=Gluster 3.8
baseurl=http://mirror.centos.org/centos/7/storage/$basearch/gluster-3.8/
gpgcheck=0
enabled=1
```
> Note: With a valid Red Hat Enterprise subscription different repositories would be used

* CTRL+O to Save. CTRL+Q to Quit.

* Install gluster storage
```bash
yum install -y glusterfs-server
```
* Enable the gluster service
```bash
systemctl enable glusterd
```

* Start the gluster service
```bash
systemctl start glusterd
```

* Add the gluster firewall exceptions
```bash
firewall-cmd --zone=public --add-port=24007-24008/tcp --permanent
firewall-cmd --zone=public --add-port=24009/tcp --permanent
firewall-cmd --zone=public --add-service=nfs --add-service=samba --add-service=samba-client --permanent
firewall-cmd --zone=public --add-port=111/tcp --add-port=139/tcp --add-port=445/tcp --add-port=965/tcp --add-port=2049/tcp --add-port=38465-38469/tcp --add-port=631/tcp --add-port=111/udp --add-port=963/udp --add-port=49152-49251/tcp --permanent
firewall-cmd --reload
```

* Enable http to use gluster in SELinux
```
setsebool -P httpd_use_fusefs 1
```

* On the first server, execute the probe command point to the second server
```bash
gluster peer probe 10.0.0.5
```

* On the second server, execute the probe command point to the first server
```bash
gluster peer probe 10.0.0.4
```

* Create a new gluster volume and start (Execute only or server 1)
```bash
gluster volume create volume1 replica 2 transport tcp 10.0.0.4:/datadrive 10.0.0.5:/datadrive force
gluster volume start volume1
```

* On the first server, Mount the gluster volume, point to the second server
```bash
mkdir /var/www/ftdemo
mount -t glusterfs 10.0.0.5:/volume1 /var/www/ftdemo
```

* And, still on the first server, add the following line to ***/etc/fstab*** for automatic mount
```
10.0.0.5:/volume1 /var/www/ftdemo glusterfs defaults,_netdev 0 0
```

* On the second server, Mount the gluster volume, point to the first server
```bash
mkdir /var/www/ftdemo
mount -t glusterfs 10.0.0.4:/volume1 /var/www/ftdemo
```

* And, still on the second server, add the following line to ***/etc/fstab*** for automatic mount
```
10.0.0.4:/volume1 /var/www/ftdemo glusterfs defaults,_netdev 0 0
```



# Reconfigure Apache

* Execute the following steps on both Web Servers
* Install additional packages for apache
```bash
yum install -y php php-common php-mysql php-gd php-xml php-mbstring php-mcrypt
```

* Enable selinux access to database
```bash
setsebool -P httpd_can_network_connect_db=1
```

* Create a new apache configuration file
```bash
nano /etc/httpd/conf.d/ftdemo.conf
```

* Include de following configuration on the new file
```
<VirtualHost *:80>
    ServerName <DNS NAME>.westus2.cloudapp.azure.com
    DocumentRoot /var/www/ftdemo
</VirtualHost>
```
* CTRL+O to Save. CTRL+Q to Quit.

* Restart apache
```bash
systemctl restart httpd
```

# Prepare Wordpress Installation 

* Execute these steps only on ***Web Server 1***
* Download the latest version of Wordpress and copy contents to the final location
```bash
cd /tmp
wget http://wordpress.org/latest.tar.gz
tar xzf latest.tar.gz
mv wordpress/* /var/www/ftdemo
```
* Give permissions to apache
```
chown -R apache:apache /var/www/ftdemo
```

* Open the wordpress configuration file
```
cd /var/www/ftdemo
cp wp-config-sample.php wp-config.php
nano wp-config.php
```

* Change the mysql settings. 
```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'ftdemo');
/** MySQL database username */
define('DB_USER', 'ftdemodbuser');
/** MySQL database password */
define('DB_PASSWORD', '<Password>');
/** MySQL hostname */
define('DB_HOST', '<Azure Internal Load Balancer IP>');
```

# Install wordpress
  * Browse to the load balancer public IP dns: **http://(prefix).westus2.cloudapp.azure.com/**
  * You will see the WordPress Welcome Page
  * Fill out the form and click ***Install Wordpress***

   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-16.png)

   * When the installation is completed a success page will appear
   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-23.png)

   * Click ***Log in**, enter your credentials and ensure you can access the Wordpress backoffice
   ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-24.png)


# Testing
 * Browse to the load balancer public IP dns **http://(prefix).westus2.cloudapp.azure.com/**
 ![Screenshot](media/website-on-iaas-http-linux/linuxpoc-25.png)

