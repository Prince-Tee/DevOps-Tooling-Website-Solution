# **DevOps Tooling Website Solution**

This documentation outlines the steps to implement a DevOps tooling website solution using a **LAMP stack**, configured with a remote **MySQL Database Server** and **NFS** (Network File System) for shared storage. The setup involves the creation of **3 Web Servers** connected to the shared NFS storage, a database, and web tooling application deployment.

## **Table of Contents**
- [Step 1: Prepare NFS Server](#step-1-prepare-nfs-server)
- [Step 2: Configure the Database Server](#step-2-configure-the-database-server)
- [Step 3: Prepare Web Servers](#step-3-prepare-web-servers)
- [Step 4: Deploy Tooling Application](#step-4-deploy-tooling-application)

---

## **Step 1: Prepare NFS Server**

### **Launch an EC2 Instance**

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/create%20an%20instnce%20on%20aws.PNG)

1. **Launch a new EC2 instance** on AWS with the **RHEL 8** operating system. Ensure it is configured with the necessary security groups to allow SSH access (port 22) and NFS communication.

2. **SSH into your NFS instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<NFS-Server-Public-IP-Address>
   ```
![sreenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/ssh%20into%20your%20instance.PNG)

### **Configure LVM on NFS Server**

Following your LVM experience from previous projects, proceed with the following steps:

1. **Install necessary LVM tools:**

   ```bash
   sudo yum install lvm2 -y
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/installing%20lvm2%20package%20to%20able%20to%20create%20physical%20volume.PNG)
2. **List attached storage devices:**

   ```bash
   lsblk
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/lsblk%20using%20it%20to%20list%20the%20attached%20volumes.PNG)

3. **Create physical volumes (PVs):**

   ```bash
   sudo pvcreate /dev/nvme1n1 /dev/nvme2nl /dev/nvme3n1
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/physical%20volume%20created.PNG)
4. **Create a volume group (VG):**

   ```bash
   sudo vgcreate webdata-vg /dev/nvme1n1 /dev/nvme2nl /dev/nvme3n1
   ```
    ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/creating%20volume%20group%20and%20logical%20volume.PNG)

5. **Create logical volumes:**

   ```bash
   sudo lvcreate -L 5G -n lv-apps webdata-vg
   sudo lvcreate -L 5G -n lv-logs webdata-vg
   sudo lvcreate -L 5G -n lv-opt webdata-vg
   ```
    ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/creating%20volume%20group%20and%20logical%20volume.PNG)
6. **Format logical volumes as XFS:**

   ```bash
   sudo mkfs.xfs /dev/webdata-vg/lv-apps
   sudo mkfs.xfs /dev/webdata-vg/lv-logs
   sudo mkfs.xfs /dev/webdata-vg/lv-opt
   ```
    ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/format%20the%20logical%20volume%20using%20xfs%20-%20Copy.PNG)

### **Mount Logical Volumes**

1. **Create mount points:**

   ```bash
   sudo mkdir -p /mnt/apps /mnt/logs /mnt/opt
   ```
     ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/creating%20mount%20for%20the%20logical%20volume.PNG)

2. **Mount the logical volumes:**

   ```bash
   sudo mount /dev/webdata-vg/lv-apps /mnt/apps
   sudo mount /dev/webdata-vg/lv-logs /mnt/logs
   sudo mount /dev/webdata-vg/lv-opt /mnt/opt
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/mount%20the%20logiacl%20volume%20to%20the%20new%20mount%20directories.PNG)

**Make Mounts Persistent Across Reboots
To make sure the mounts persist after reboot, you need to add them to the /etc/fstab file**:


sudo vi /etc/fstab
Add the following lines:

```bash

/dev/vg_nfs/lv-apps /mnt/apps xfs defaults 0 0
/dev/vg_nfs/lv-logs /mnt/logs xfs defaults 0 0
/dev/vg_nfs/lv-opt /mnt/opt xfs defaults 0 0  
``` 
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/content%20of%20%20sudo%20vi%20etc%20fstab.PNG)
3. **Verify the mounts:**

   ```bash
   df -h
   ```

### **Install and Configure NFS Server**

1. **Install NFS utilities:**

   ```bash
   sudo yum install nfs-utils -y
   ```
   ![screenshots](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/Install%20the%20necessary%20NFS%20packages.PNG)

2. **Start and enable NFS server:**

   ```bash
   sudo systemctl start nfs-server.service
   sudo systemctl enable nfs-server.service
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/starting%20the%20NFS%20server.PNG)

3. **Verify NFS status:**

   ```bash
   sudo systemctl status nfs-server.service
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/starting%20the%20NFS%20server.PNG)

### **Configure NFS Exports**

1. **Set ownership and permissions on the mount points:**

   ```bash
   sudo chown -R nobody: /mnt/apps /mnt/logs /mnt/opt
   sudo chmod -R 777 /mnt/apps /mnt/logs /mnt/opt
   ```

    ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/set%20the%20ownership%20of%20the%20mounted%20directories.PNG)

  ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/Set%20the%20permissions%20to%20allow%20reading%2C%20writing%2C%20and%20execution.PNG)

2. **Export the directories for the web servers’ subnet (replace `<Subnet-CIDR>` with the actual subnet CIDR):**
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/the%20ipv4%20cidr.PNG)

   ```bash
   sudo vi /etc/exports
   ```

   Add the following lines:

   ```bash
   /mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   /mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   /mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/content%20of%20the%20sudo%20vi%20%20etc%20exports.PNG)
3. **Apply the export settings:**

   ```bash
   sudo exportfs -arv
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/applying%20exports.PNG)

4. **Check which ports are used by NFS:**

   ```bash
   rpcinfo -p | grep nfs
   ```
   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/checking%20which%20nfs%20is%20using.PNG)

5. **Open NFS ports (TCP 111, UDP 111, UDP 2049) on the NFS security group.**

---
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/configuring%20portrange%20on%20security%20group%20on%20instance.PNG)

## **Step 2: Configure the Database Server**

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20create%20your%20database%20server.PNG)
1. **Launch a new EC2 instance with RHEL 8 for the database server.**

2. **SSH into the Database instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<Database-Server-Public-IP-Address>
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20ssh%20into%20the%20database%20server.PNG)

3. **Install MySQL server:**

   ```bash
   sudo yum install mysql-server -y
   sudo systemctl start mysqld
   sudo systemctl enable mysqld
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20updating%20and%20installing%20mysql.PNG)

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20starting%20mysql.PNG)

4. **Secure MySQL installation:**

   ```bash
   sudo mysql_secure_installation
   ```
  ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20%20sudo%20mysql%20secure%20installation.PNG)
  
5. **Log in to MySQL:**

   ```bash
   mysql -u root -p
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20Log%20into%20MySQL%20as%20root.PNG)

6. **Create the `tooling` database and user `webaccess`:**

   ```sql
   CREATE DATABASE tooling;
   CREATE USER 'webaccess'@'<Subnet-CIDR>' IDENTIFIED BY 'password';
   GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'<Subnet-CIDR>';
   FLUSH PRIVILEGES;
   ```

---
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/DB%20Create%20the%20webaccess%20user%20and%20grant%20it%20permissions%20on%20the%20tooling%20database.PNG)

## **Step 3: Prepare the Web Servers**

1. **Launch 3 new EC2 instances with RHEL 8 for the web servers.**

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20creating%20three%20webservers.PNG)

2. **SSH into each web server instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<Web-Server-Public-IP-Address>
   ```

   ![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20ssh%20into%20all%20your%20three%20webservers.PNG)

3. **Install NFS client and mount shared storage:**

   ```bash
   sudo yum install nfs-utils nfs4-acl-tools -y
   sudo mkdir -p /var/www
   sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
   ```
   ![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20installing%20nfs%20client%20on%20all%20three%20webservers.PNG)

   ![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20creating%20sudo%20mkdir%20varwww%20on%20all%20the%20three%20webservers.PNG)

4. **Make NFS mount persistent after reboot:**


Create the /var/www directory on each web server (if not already present):

```bash
sudo mkdir /var/www
```
Mount the NFS share to the /var/www directory:
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20Mount%20the%20NFS%20share%20to%20the%20var%20www%20directory.PNG)

   ```bash
   sudo vi /etc/fstab
   ```

   Add the following line:

   ```bash
   <NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
   ```
   ![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20content%20of%20the%20sudo%20vi%20etcfstab%20Make%20the%20mount%20permanent.PNG)

5. **Verify the mount:**

   ```bash
   df -h
   ```
   ![screeshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20verify%20that%20mount%20is%20working%20on%20all%20the%20web%20server.PNG)

### **Install Apache and PHP on Web Servers**

1. **Install Apache and PHP:**

   ```bash
   sudo yum install httpd -y
   sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
   sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
   sudo dnf module reset php
   sudo dnf module enable php:remi-7.4
   sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
   ```
![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20insall%20apache%20sudo%20yum%20install%20httpd%20-y.PNG)

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/db%20Install%20Remi's%20repository%20and%20PHP.PNG)
2. **Start and enable Apache:**

   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

---

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20Start%20and%20enable%20Apache%20on%20all%20the%20three%20web%20servers.PNG)

**Start and enable PHP-FPM**:

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20Start%20and%20enable%20PHP-FPM%20on%20all%20three%20web%20server.PNG)

Disable SELinux (Optional)
Temporarily disable SELinux to avoid permission issues

```bash

sudo setenforce 0
```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20Temporarily%20disable%20SELinux%20to%20avoid%20permission%20issues.PNG)

To disable SELinux permanently, edit the config file:

```bash

sudo vi /etc/sysconfig/selinux
```
Change the line:

```ini
SELINUX=enabled
```

To:

```ini

SELINUX=disabled
```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20disable%20SELinux%20permanently%20edit%20the%20config%20file.PNG)

Restart Apache:


```bash

sudo systemctl restart httpd
```

![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20Restart%20Apache.PNG)

## **Step 4: Deploy Tooling Application**

1. **Fork the tooling source code from the StegHub GitHub account to your GitHub account.**

https://github.com/StegTechHub/tooling.git 

2. **Clone the repository onto your web servers:**

   ```bash
   git clone <Your-GitHub-Repo-URL>
   ```
   ![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20cd%20var%20www%20html%20then%20install%20git.PNG)
3. **Deploy the website to `/var/www/html`:**

   ```bash
   sudo cp -r /path/to/repo/html/* /var/www/html/
   ```
![screesnhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20cd%20var%20www%20html%20then%20install%20git.PNG)

4. **Update the `functions.php` file to connect to the MySQL database by specifying the correct database credentials.**
![screenhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/wb%20%20Update%20the%20database%20connection%20setting%20functions%20php.PNG)
5. **Apply the `tooling-db.sql` script to the database:**

   ```bash
   mysql -h <Database-Private-IP> -u webaccess -p tooling < tooling-db.sql
   ```

6. **Create a new admin user in MySQL:**

   ```sql
   INSERT INTO 'users' ('id', 'username', 'password', 'email', 'user_type', 'status') VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
   ```

---
![screenhot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/creating%20webaccess%20a%20my%20user.PNG)

## **Verification**

1. **Access the website

** via the web servers’ public IP address:

   ```bash
   http://<Web-Server-Public-IP>/index.php
   ```
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/final%20output.PNG)

You should see the tooling website. Confirm that you can log in with the admin credentials .
![screenshot](https://github.com/Prince-Tee/DevOps-Tooling-Website-Solution/blob/main/screenshots%20from%20my%20local%20env/access%20my%20web%20server.PNG)


---

This completes the **DevOps Tooling Website Solution** setup with **3 web servers**, NFS, and a remote MySQL database. You can now begin monitoring and improving this infrastructure.

---