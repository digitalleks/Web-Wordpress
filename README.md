# Web-Wordpress
Web Solution with Wordpress

This projects starts with the preparation of the web server on AWS.
Launch an EC2 instance that will serve as "Web Server".\
<img width="716" alt="AWS Webserver EC2" style="border:3px solid #66CC33" src="https://user-images.githubusercontent.com/61512079/177868441-57c579a7-a92b-451d-8596-87f5237faff1.PNG">

Create 3 EBS Volume and attach each of them to the Webserver Instance:\
<img width="766" alt="3-Volume-created" src="https://user-images.githubusercontent.com/61512079/177868598-8a1134b7-f733-4de3-b6cd-700e89074d76.PNG">\
<img width="492" alt="vol-attached1" src="https://user-images.githubusercontent.com/61512079/177868672-caa0b29a-359d-4239-93d1-6dc71e3c672d.PNG">\
<img width="492" alt="vol-attached2" src="https://user-images.githubusercontent.com/61512079/177868699-9d9e18ee-f969-4ee7-87b4-16d6e3abb489.PNG">\
<img width="494" alt="vol-attached3" src="https://user-images.githubusercontent.com/61512079/177868743-f6d5bf61-11b4-4248-8721-159608b9e788.PNG">\

All the created blocks are confirmed with the command below:
```bash
lsblk
```
Output\
<img width="301" alt="block-confirm" src="https://user-images.githubusercontent.com/61512079/177872560-cdb49cf1-d22f-4517-ba9e-97683d47642f.PNG">

The free space on the created device is confirmed as follow:\
<img width="301" alt="block-confirm" src="https://user-images.githubusercontent.com/61512079/177872893-ad987648-dd72-49d2-91d5-caeaf711cb82.PNG">

The gdisk utility is used to create a single partition on each of the 3 disks as follow:
```bash
sudo gdisk /dev/xvdf
```
Output of the partition creation are shown below:\
<img width="254" alt="partition" src="https://user-images.githubusercontent.com/61512079/177877199-de848ac6-ea9b-4afe-8b47-cf6b39039196.PNG">

Next is the installation of lvm2 to enable us check the available partition:
```bash
 sudo yum install lvm2
```
<img width="287" alt="avalable-partition" src="https://user-images.githubusercontent.com/61512079/177877634-ed07b1a5-e8d2-48bd-8abb-20214766942d.PNG">

Then we use **pvcreate** utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM:
```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
Output of pvcreate on each of the disk:\
<img width="336" alt="pvcreate" src="https://user-images.githubusercontent.com/61512079/177878056-be1ca9f5-ba4d-4c4d-8ee1-94af5995f41a.PNG">\
<img width="254" alt="pv-verify" src="https://user-images.githubusercontent.com/61512079/177878186-b17c10b4-4862-43b7-b7a1-36e2a84582bf.PNG">

Next the **vgcreate** utility is used to add all 3 PVs to a volume group (VG) named  **webdata-vg**:
```bash
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```
<img width="528" alt="vgcreate" src="https://user-images.githubusercontent.com/61512079/177878530-d8833d81-8bc0-4454-929e-d55a5a0fa336.PNG">
The volume group is verified as follow:

```bash
sudo vgs
```
<img width="316" alt="volume-verify" src="https://user-images.githubusercontent.com/61512079/177878806-2c44e130-207f-491a-a86e-ba86d91767f4.PNG">

Two logical volumes are created using the **lvcreate** command: **apps-lv**  to store data for the Website and **logs-lv** to store data for logs.
```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
Logical volume created is verified below:\
<img width="546" alt="lvs" src="https://user-images.githubusercontent.com/61512079/177879849-6f0d8829-5fec-4f5c-b70b-5492796c8384.PNG">

The entire setup is verified using the commands below:
```bash
sudo vgdisplay -v #view complete setup - VG, PV, and LV
sudo lsblk 
```
Output:\
<img width="578" alt="VG-LV" src="https://user-images.githubusercontent.com/61512079/177880348-9bea87c6-39b2-40a2-ac5f-89e6e79c2efb.PNG">\
<img width="581" alt="LV-PV" src="https://user-images.githubusercontent.com/61512079/177880314-6ec5cbaa-ac19-48c5-9c0b-0be7dee10502.PNG">\
<img width="371" alt="entire-disk-setup" src="https://user-images.githubusercontent.com/61512079/177880390-96d56a53-7606-4814-9956-3a9335379185.PNG">


Next we used **mkfs.ext4** to format the logical volumes with ext4 filesystem:
```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
Output of ext4 formating:\
<img width="486" alt="ext4_apps_lv" src="https://user-images.githubusercontent.com/61512079/177880765-457b7621-df9f-4df3-9d4d-c8b415710ebe.PNG">\
<img width="490" alt="ext4_logs_lv" src="https://user-images.githubusercontent.com/61512079/177880786-69170c54-3f40-474c-a666-afd659214b13.PNG">

Then /var/www/html directory is created to store website files as follow:
```bash
sudo mkdir -p /var/www/html
```
And /home/recovery/logs directory is also created to store backup of log data:
```bash
sudo mkdir -p /home/recovery/logs
```
Then **/var/www/html** is mounted on **apps-lv** logical volume as follow:
```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
The  **rsync** utility is used to backup all the files in the log directory **/var/log** into **/home/recovery/logs** as a prerequisite to mounting the logs volume:
```bash
sudo rsync -av /var/log/. /home/recovery/logs/
```
Then **/var/log** is mounted on **logs-lv** logical volume.
```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```
Next the backup logs is restored back into /var/log directory:
```bash
sudo rsync -av /home/recovery/logs/. /var/log
```
Finally, the  /etc/fstab file is updated so that the mount configuration will persist after restart of the server.:
```bash
sudo blkid
```
The /etc/fstab always starts as read-only file, so to modify it, run the command below before editing it:
```bash
sudo mount -n -o remount /
```
Then:
```bash
vi /etc/fstab
```
Entries modified as show:/
<img width="883" alt="fstab edit" src="https://user-images.githubusercontent.com/61512079/177884374-2e66f16c-b47b-45f9-8cc3-a9eb40741096.PNG">

Issue the commands below to test the configuration and reload the daemon:
```bash
sudo mount -a
sudo systemctl daemon-reload
```
The whole disk setup is verified as shown below:\
<img width="449" alt="setup-verify" src="https://user-images.githubusercontent.com/61512079/177884700-1debf672-a58d-41dd-957a-d7dbdcdc3c9f.PNG">

Below is the part 2 of the project which is the Database instance setup:
The disk is also prepared using the same steps as part 1 except that the server name is **db-server** and mount is created to /db directory.
The output of the disk preparation is shown below:\
<img width="258" alt="DB-partition" src="https://user-images.githubusercontent.com/61512079/177888263-38b1936a-a04c-4097-a2b5-774771fddba0.PNG">\
<img width="258" alt="DB-PVCREATE" src="https://user-images.githubusercontent.com/61512079/177888286-8a333ed8-c1c0-4401-a09a-6c7036b925a0.PNG">\
<img width="458" alt="db-lvcreate" src="https://user-images.githubusercontent.com/61512079/177888313-a3e2b8af-ea00-4ffa-8160-c76c7e920bf6.PNG">\
<img width="548" alt="db-vgcreate" src="https://user-images.githubusercontent.com/61512079/177888339-6445538b-1d9d-4c5f-a084-4941d4667ba9.PNG">\
<img width="506" alt="DB-ext4-format" src="https://user-images.githubusercontent.com/61512079/177888374-26b63795-ce41-48a7-b03b-fa6393a99859.PNG">

The partitions are mounted on the created directories and verified as follow:\
<img width="481" alt="db-mount-verify" src="https://user-images.githubusercontent.com/61512079/177890236-854f55ed-537f-4342-b436-782e9def12cc.PNG">

The third part of the project is the installation of wordpress on the EC2 Webserver as follow:
```bash
sudo yum -y update
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
Then Apache is started as follow:
```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```
Next install PHP and its dependencies:
```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
```
After the PHP dependencies installation, Apache is restarted:
```bash
sudo systemctl restart httpd
```
The next step is the installation of wordpress:
```bash
  mkdir wordpress
  cd   wordpress
  sudo wget http://wordpress.org/latest.tar.gz
  sudo tar xzvf latest.tar.gz
  sudo rm -rf latest.tar.gz
  sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
  sudo cp -R wordpress /var/www/html/
```
Next is configuration of  SELinux Policies
```bash
  sudo chown -R apache:apache /var/www/html/wordpress
  sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
  sudo setsebool -P httpd_can_network_connect=1
```

The 4th step is the installation of MySQL on the DB Server EC2 Instance:
```bash
sudo yum update
sudo yum install mysql-server
```
The service is started and verified up as shown:
```bash
sudo systemctl restart mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```
MySQL status below:\
<img width="727" alt="mysql-status-dbserver" src="https://user-images.githubusercontent.com/61512079/177892067-3f94a906-5d87-47e3-b105-c41f483cfefc.PNG">

Next step is to configure the DB to work with WOrdpress:
```bash
sudo mysql
```
```mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```
Output of the database creation  is shown here:\
<img width="432" alt="Db-wordpress-user" src="https://user-images.githubusercontent.com/61512079/177893795-7e11935d-1634-43a8-b5c7-2312c4233e86.PNG">

On the DB Server Instance, the security group is opened to allow port 3306:\
<img width="680" alt="DB-Port-allow" src="https://user-images.githubusercontent.com/61512079/177893179-368c99fb-32cb-4a27-8c44-996a63a17519.PNG">

The Wordpress was able to connect to the DB Server using the mysql utility:\
<img width="706" alt="Db-wordpress-connect-test" src="https://user-images.githubusercontent.com/61512079/177893956-b2e01476-5a56-4e5b-ad57-afa771d8134a.PNG">

Next permission and configuration  is set for Apache to use Wordpress in the steps following:

In the Apache config file directory(**/etc/httpd/conf.d**), create a config file for the new wordpress site with:
```bash
sudo touch wordpress.conf
sudo chmod 755 wordpress.conf
sudo vi wordpress.conf
```

Create virtual host for the wordpress file and paste the config below in it:
```bash
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName wordpress.dummyhost.com
    DocumentRoot /var/www/html/wordpress
    ErrorLog /var/log/apache_error.log
    CustomLog /var/log/access.log combined
</VirtualHost>
<Directory /var/www/html/wordpress>
Require all granted
AllowOverride All
</Directory>
</VirtualHost>
```

Start the httpd service and enable it on startup and create a firewall rule for httpd using the following commands:
```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/var/www/html/wordpress(/.*)?'
sudo restorecon -Rv /var/www/html/wordpress
```

In order to run the firewalld command, the program is install using the command below:
```bash
sudo yum install firewalld
```

After installing the firewall program, start the firewalld process with the commands below:
```bash
 sudo systemctl start firewalld
 sudo systemctl start firewalld
 sudo systemctl enable firewalld
 ```
<img width="920" alt="Apache firewall config" src="https://user-images.githubusercontent.com/61512079/180576622-dde2850d-5137-4e38-a571-00c0a40dbbf8.PNG">

Once the program is confirmed running, you can run this commands to create the firewall rule for the httpd as follow:
```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```
In order to test the wordpress site from your PC, the hosts file was edit to use the dummy host name created as follow:
Run WIndows command prompt as Administrator to be able to edit the hosts file.
```windows
C:\Windows\System32\drivers\etc>notepad hosts
```
Add the entry below -
3.16.54.201 wordpress.dummyhost.com
press ctrl+s to save

Test by opening http://wordpress.dummyhost.com/ on the browser: 
The output is as shown below:
<img width="582" alt="Wordpress Page test" src="https://user-images.githubusercontent.com/61512079/180576507-218e2327-a8e6-44e5-b09c-7876919f43b6.PNG">





