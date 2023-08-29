# Project-6
WEB SOLUTION WITH WORDPRESS

**In this project, we will have the hands-on experience that showcases Three-tier Architecture while also ensuring that the disks used to store files on the Linux servers are adequately partitioned and managed through programs such as gdisk and LVM respectively.**

**We will be working working with several storage and disk management concepts.**

## LAUNCH AN EC2 INSTANCE THAT WILL SERVE AS “WEB SERVER”.

### Step 1 Prepare the Web Server

1. Launch an EC2 instance that will serve as “Web Server”. Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.

![NGINX Status](./Images-6/Image-6.1.jpg)



2. Attach all three volumes one by one to your Web Server EC2 instance

![NGINX Status](./Images-6/Image-6.2.jpg)


**Open up the Linux terminal to begin configuration**

3. Use `lsblk` command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there – their names will likely be xvdf, xvdh, xvdg.

![NGINX Status](./Images-6/Image-6.3.jpg)

4. Use `df -h` command to see all mounts and free space on your server

* Use `gdisk` utility to create a single partition on each of the 3 disks

`sudo gdisk /dev/xvdf`

![NGINX Status](./Images-6/Image-6.4.jpg)

5. Use `lsblk` utility to view the newly configured partition on each of the 3 disks.

![NGINX Status](./Images-6/Image-6.5.jpg)

6. Install `lvm2` package using `sudo yum install lvm2`. Run `sudo lvmdiskscan` command to check for available partitions.

![NGINX Status](./Images-6/Image-6.6.jpg)

* **In Ubuntu, we used `apt` command to install packages, in RedHat/CentOS we use `yum` which is a different package manager.**

7. Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

8. Verify that your Physical volume has been created successfully by running `sudo pvs`

![NGINX Status](./Images-6/Image-6.7.jpg)

9. Use` vgcreate` utility to add all 3 PVs to a volume group (VG). Name the VG **webdata-vg**

`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

10. Verify that your VG has been created successfully by running `sudo vgs`

![NGINX Status](./Images-6/Image-6.8.jpg)

11. Use `lvcreate` utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.

```
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```

12. Verify that your Logical Volume has been created successfully by running `sudo lvs`

![NGINX Status](./Images-6/Image-6.9.jpg)

13. Verify the entire setup, Run;
```
sudo vgdisplay -v #view complete setup - VG, PV, and LV
sudo lsblk
```
![NGINX Status](./Images-6/Image-6.10.jpg)

* Use `mkfs.ext4` to format the logical volumes with `ext4` filesystem

```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

15. Create **/var/www/html** directory to store website files
`sudo mkdir -p /var/www/html`

16. Create **/home/recovery/logs** to store backup of log data
`sudo mkdir -p /home/recovery/logs`


17. Mount **/var/www/html** on **apps-lv** logical volume
`sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

18. Use `rsync` utility to backup all the files in the log directory **/var/log** into **/home/recovery/logs** (This is required before mounting the file system)
`sudo rsync -av /var/log/. /home/recovery/logs/`

19. Mount **/var/log** on **logs-lv** logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very important)
`sudo mount /dev/webdata-vg/logs-lv /var/log`

20. Restore log files back into **/var/log** directory
`sudo rsync -av /home/recovery/logs/. /var/log`

21. Update `/etc/fstab` file so that the mount configuration will persist after restart of the server.

The UUID of the device will be used to update the `/etc/fstab` file;

`sudo blkid`

![NGINX Status](./Images-6/Image-6.11.jpg)

`sudo vi /etc/fstab`

Update `/etc/fstab` in this format using your own UUID and rememeber to remove the leading and ending quotes

![NGINX Status](./Images-6/Image-6.12.jpg)

22. Test the configuration and reload the daemon

```
sudo mount -a

sudo systemctl daemon-reload
```
23. Verify your setup by running `df -h`, output must look like this:

![NGINX Status](./Images-6/Image-6.13.jpg)

## Step 2 — Prepare the Database Server
Launch a second RedHat EC2 instance that will have a role – ‘DB Server’
Repeat the same steps as for the Web Server, but instead of `apps-lv` create `db-lv` and mount it to `/db` directory instead of `/var/www/html/`.

## Step 3 — Install WordPress on your Web Server EC2
* Update the repository
`sudo yum -y update`

* Install wget, Apache and it’s dependencies

`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

* Start Apache
```
sudo systemctl enable httpd
sudo systemctl start httpd
```
* To install PHP and it’s dependencies

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo dnf module enable php:8.1
sudo dnf module -y install php:8.1/common
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
```
* Restart Apache

`sudo systemctl restart httpd`

* **Download wordpress and copy wordpress to `var/www/html`**

```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
sudo cp -R wordpress /var/www/html/
```
* Configure SELinux Policies
```
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```

## Step 4 — Install mysql on DB Server EC2

```
sudo yum update
sudo yum install mysql-server
```
Let verify that the service is up and running by using `sudo systemctl status mysqld`, if it is not running, restart the service and enable it so it will be running even after reboot

```
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

## Step 5 — Configure DB to work with WordPress

```
sudo mysql
CREATE DATABASE wordpress;
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'%' IDENTIFIED WITH mysql_native_password BY 'wordpress';
 GRANT ALL PRIVILEGES ON *.* TO 'wordpress'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

**See all users and hosts**
`select user, host from mysql.user; `

`SHOW DATABASES;`

![NGINX Status](./Images-6/Image-6.14.jpg)

`exit`


* **Configure MySQL server to allow connections from remote hosts i.e set the bind address** 

`sudo vi /etc/my.cnf`

![NGINX Status](./Images-6/Image-6.17.jpg)

 **Restart the service**

`sudo systemctl restart mysqld`

## Step 6 — Configure WordPress to connect to remote database.


Hint: Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32.  **OR OPEN ALL TRAFFIC**

![NGINX Status](./Images-6/Image-6.15.jpg)

**Next you disable the apache default web page**

`sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup`

1. Install MySQL client and test that you can connect from your Web Server to your DB server by using `mysql-client`


`sudo yum install mysql`

`sudo mysql -u wordpress -p -h <DB server public address>`

2. Verify if you can successfully execute` SHOW DATABASES;` command and see a list of existing databases.

![NGINX Status](./Images-6/Image-6.16.jpg)

3. Change permissions and configuration so Apache could use WordPress:

`sudo vi /var/www/html/wordpress/wp-config.php`


 * **Configure** `DB_NAME`, `DB_USER`, `DB_PASSWORD`, and `DB_HOST`

 `sudo systemctl restart httpd`


![NGINX Status](./Images-6/Image-6.18.jpg)


`sudo systemctl restart php-fpm`

4. Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)

5. Try to access from your browser the link to your WordPress `http://<Web-Server-Public-IP-Address>/wordpress/`

![NGINX Status](./Images-6/Image-6.19.jpg)
 
Fill out your DB credentials:

![NGINX Status](./Images-6/Image-6.20.jpg)
