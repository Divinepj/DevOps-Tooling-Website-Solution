# DevOps-Tooling-Website-Solution
In previous project [Implementing Web Solution https://github.com/Divinepj/Implementing-Wordpress-Web-Solution] I implemented a WordPress based solution that is ready to be filled with content and can be used as a full fledged website or blog. Moving further I will add some more value to my solution so that a member of a DevOps team could utilize.

In this project,, I will be introducing the concept of file sharing for multiple servers to share the same web content and also a database for storing data related to the website using **RHEL-8.6.0_HVM-20220503-x86_64-2-Hourly2-GP2** 
![alt text](./IMG/Snipaste_2023-11-09_17-49-57.png)

## Preparing NFS Server
Create an EC2 instance (Red Hat Enterprise Linux 8 on AWS) on which we will setup our NFS(Network File Storage) Server.

On this server we attach 2 EBS volumes 10GB each as external storage to our instance and create 3 logical volumes on it through which we will attach mounts from our external web servers.
![alt text](./IMG/Snipaste_2024-04-26_18-03-52.png)

-   To be able to create logical volumes lv we have to create physical volumes on the partitions created to do this we need to install the lvm2 package by running sudo yum install lvm2 as shown below 
![](./IMG/Snipaste_2024-04-26_18-04-18.png)
we then proceed to create 3 logical volumes lv-opt, lv-apps and lv-logs as shown below
![alt text](./IMG/Snipaste_2024-04-26_18-07-10.png)

We then create the `xfs` file systems as shown below
![alt text](./IMG/Snipaste_2023-11-09_18-29-56.png) and ![alt text](./IMG/Snipaste_2023-11-09_18-27-31.png)

we therefore proceed to create directories `/mnt/apps`, `/mnt/opt`, and `/mnt/logs`
![alt text](./IMG/Snipaste_2023-11-09_18-34-15.png)
we then mount this on the logical volumes as shown below ![alt text](./IMG/Snipaste_2023-11-09_18-37-02.png)

we then proceed to install the NFS server by running the following commands 
```
{
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
}
```
![alt text](./IMG/Snipaste_2023-11-09_18-45-25.png)
![alt text](./IMG/Snipaste_2023-11-09_18-46-40.png)

we then make sure that the the permissions that will allow our web servers to read, write and execute file on NFS by running the commands below 
```
{
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
}
```

![alt text](./IMG/Snipaste_2023-11-09_18-52-31.png)

we then configure access to NFS for clients within the same subnet CIDR. we get the subnet cidr on the AWS web console as by running the command below `sudo vi /etc/exports` shown below 
![alt text](./IMG/Snipaste_2023-11-09_19-02-28.png)
we add the following to the configuration file as shown below
```
{
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
}
```

![alt text](./IMG/Snipaste_2023-11-09_19-03-52.png)

and `sudo exportfs -arv` as shown below ![alt text](./IMG/Snipaste_2023-11-09_19-05-05.png)

we then check the which port is used by NFS and open it using security groups by running the commands ![alt text](./IMG/Snipaste_2023-11-09_19-05-55.png)

![alt text](./IMG/Snipaste_2023-11-09_19-12-48.png)

## Preparing Database Server

Create an Redhat Server on AWS which will serve as our Database. Ensure its in the same subnet as the NFS-Server.

we then proceed to Install mysql-server by running the commands below 
```
{
sudo apt -y update
sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
sudo systemctl status mysql
}
```
![alt text](./IMG/Snipaste_2023-11-09_19-18-06.png)

![alt text](./IMG/Snipaste_2023-11-09_19-19-49.png)

We then create a database and name it tooling, create a database user and name it webaccess and grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr by running the commands 
```
{
sudo mysql
CREATE DATABASE tooling;
CREATE USER `webaccess`@`<subnet cidr>` IDENTIFIED BY 'mypass';
GRANT ALL ON tooling.* TO 'webaccess'@'<subnet cidr>';
FLUSH PRIVILEGES;
SHOW DATABASES;
SELECT User FROM mysql.user;
}
```
as shown below ![alt text](./IMG/Snipaste_2023-11-09_19-47-18.png)

## Preparing Web Servers

Create a RHEL EC2 instance on AWS which serves as our web server. Also remember to have in it in same subnet

A couple of configurations will be done on the web servers:

- configuring NFS client
- deploying tooling website application
- configure servers to work with database
Installing NFS-Client

![alt text](./IMG/Snipaste_2023-11-09_19-53-48.png)

We will then connect our `/var/www` directory to our webserver with the `/mnt/apps` on nfs server. This is acheived by mounting the NFS server directory to the webserver directory by running ``sudo mount -t nfs -o rw,nosuid <NFS private IP>:/mnt/logs /var/log/httpd` as shown below

![alt text](./IMG/Snipaste_2023-11-09_19-56-31.png)
 

We then ensure that our mounts remain intact when the server reboots. This is achieved by configuring the fstab directory by running the `sudo vi /etc/fstab` 
add the following line <NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0 as shown below ![alt text](./IMG/Snipaste_2023-11-09_19-57-55.png)
we also mount the log folder for Apache on the webservers to NFS server's export for logs as shown below 
![alt text](./IMG/Snipaste_2024-04-27_08-45-47.png)

![alt text](./IMG/Snipaste_2024-04-27_08-50-47.png)

## Installing Apache and Php
we then run the following commands to install both Apache and PHP 

`sudo yum install httpd -y`  As shown below 
![alt text](./IMG/Snipaste_2024-04-26_20-40-46.png)

`sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm` as shown below

![alt text](./IMG/Snipaste_2024-04-26_20-41-38.png)


`sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm` as shown below

![alt text](./IMG/Snipaste_2024-04-26_20-42-33.png)


`sudo dnf module reset php` and `sudo dnf module enable php:remi-7.4` as shown below

![alt text](./IMG/Snipaste_2024-04-26_20-44-13.png)

we then run `sudo dnf install php php-opcache php-gd php-curl php-mysqlnd`

![alt text](./IMG/Snipaste_2024-04-26_20-44-58.png)
and then `sudo systemctl start php-fpm`, `sudo systemctl enable php-fpm` and `setsebool -P httpd_execmem 1` as shown below
![alt text](./IMG/Snipaste_2024-04-26_20-46-07.png)


We can see that both /var/www and /mnt/apps contains same content. This shows that both mount points are connected via NFS. ![alt text](./IMG/Snipaste_2023-11-09_20-09-57.png)
 and 
 ![alt text](./IMG/Snipaste_2023-11-09_20-10-21.png)

We then repeat these steps for 2 other web servers.

We locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Make sure the mount point will persist after reboot by running and `sudo mount -t nfs -o rw,nosuid <NFS private IP>:/mnt/logs /var/log/httpd` as shown below
![alt text](./IMG/Snipaste_2023-11-22_22-31-23.png)

We then proceed to fork the tooling source code from `Darey.io Github Account` to my github account as shown below 
![alt text](./IMG/Snipaste_2024-04-27_08-56-09.png)
Then we proceed to install git on the terminal to enable us clone the content of the forked site to our servers by running `sudo yum install git` as shown below
![alt text](./IMG/Snipaste_2024-04-27_09-00-05.png)

we also endure that the content of html folder from the repository is deployed to `html` as shown below ![alt text](./IMG/Snipaste_2024-04-27_09-18-41.png)

On the NFS Server, add web content into the /mnt/apps directory. This should contain a html folder. The same content will be present in the /var/www directory in the web server. ![alt text](./IMG/Snipaste_2023-11-09_20-55-36.png)

Run <public_ip_address>/index.php on a web browser to access the site. Use public_ip_address of the web server. TCP port 80 should be open on the web broswer as shown below ![alt text](./IMG/Snipaste_2023-11-09_20-57-04.png)

In the /var/www/html directory , edit the already written php script to connect to the database `sudo vi /var/www/html/functions.php`. After the modification , connect to the database server from the web server by running `mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql` as shown below ![alt text](/IMG/Snipaste_2024-04-27_09-42-28.png)

we then proceed to create in MYSQL a new admin user with username `myuser` and password `password` as shown below ![alt text](./IMG/Snipaste_2024-04-27_09-53-03.png)

we then proceed to login to the webserver by entering `<webserver public IP>/index.php` and login in with my user as shown below
![alt text]()



