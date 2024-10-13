Here's an updated version of the documentation with the title and other details tailored to your request. The task now highlights that three web servers are created, includes SSH instructions, and ensures clarity with detailed steps, including the minor aspects you've emphasized.

---

# **DevOps Tooling Website Solution - 101**

This documentation outlines the steps to implement a DevOps tooling website solution using a **LAMP stack**, configured with a remote **MySQL Database Server** and **NFS** (Network File System) for shared storage. The setup involves the creation of **3 Web Servers** connected to the shared NFS storage, a database, and web tooling application deployment.

## **Table of Contents**
- [Step 1: Prepare NFS Server](#step-1-prepare-nfs-server)
- [Step 2: Configure the Database Server](#step-2-configure-the-database-server)
- [Step 3: Prepare Web Servers](#step-3-prepare-web-servers)
- [Step 4: Deploy Tooling Application](#step-4-deploy-tooling-application)

---

## **Step 1: Prepare NFS Server**

### **Launch an EC2 Instance**

1. **Launch a new EC2 instance** on AWS with the **RHEL 8** operating system. Ensure it is configured with the necessary security groups to allow SSH access (port 22) and NFS communication.

2. **SSH into your NFS instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<NFS-Server-Public-IP-Address>
   ```

### **Configure LVM on NFS Server**

Following your LVM experience from previous projects, proceed with the following steps:

1. **Install necessary LVM tools:**

   ```bash
   sudo yum install lvm2 -y
   ```

2. **List attached storage devices:**

   ```bash
   lsblk
   ```

3. **Create physical volumes (PVs):**

   ```bash
   sudo pvcreate /dev/xvdf /dev/xvdg /dev/xvdh
   ```

4. **Create a volume group (VG):**

   ```bash
   sudo vgcreate webdata-vg /dev/xvdf /dev/xvdg /dev/xvdh
   ```

5. **Create logical volumes:**

   ```bash
   sudo lvcreate -L 5G -n lv-apps webdata-vg
   sudo lvcreate -L 5G -n lv-logs webdata-vg
   sudo lvcreate -L 5G -n lv-opt webdata-vg
   ```

6. **Format logical volumes as XFS:**

   ```bash
   sudo mkfs.xfs /dev/webdata-vg/lv-apps
   sudo mkfs.xfs /dev/webdata-vg/lv-logs
   sudo mkfs.xfs /dev/webdata-vg/lv-opt
   ```

### **Mount Logical Volumes**

1. **Create mount points:**

   ```bash
   sudo mkdir -p /mnt/apps /mnt/logs /mnt/opt
   ```

2. **Mount the logical volumes:**

   ```bash
   sudo mount /dev/webdata-vg/lv-apps /mnt/apps
   sudo mount /dev/webdata-vg/lv-logs /mnt/logs
   sudo mount /dev/webdata-vg/lv-opt /mnt/opt
   ```

3. **Verify the mounts:**

   ```bash
   df -h
   ```

### **Install and Configure NFS Server**

1. **Install NFS utilities:**

   ```bash
   sudo yum install nfs-utils -y
   ```

2. **Start and enable NFS server:**

   ```bash
   sudo systemctl start nfs-server.service
   sudo systemctl enable nfs-server.service
   ```

3. **Verify NFS status:**

   ```bash
   sudo systemctl status nfs-server.service
   ```

### **Configure NFS Exports**

1. **Set ownership and permissions on the mount points:**

   ```bash
   sudo chown -R nobody: /mnt/apps /mnt/logs /mnt/opt
   sudo chmod -R 777 /mnt/apps /mnt/logs /mnt/opt
   ```

2. **Export the directories for the web servers’ subnet (replace `<Subnet-CIDR>` with the actual subnet CIDR):**

   ```bash
   sudo vi /etc/exports
   ```

   Add the following lines:

   ```bash
   /mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   /mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   /mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
   ```

3. **Apply the export settings:**

   ```bash
   sudo exportfs -arv
   ```

4. **Check which ports are used by NFS:**

   ```bash
   rpcinfo -p | grep nfs
   ```

5. **Open NFS ports (TCP 111, UDP 111, UDP 2049) on the NFS security group.**

---

## **Step 2: Configure the Database Server**

1. **Launch a new EC2 instance with RHEL 8 for the database server.**

2. **SSH into the Database instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<Database-Server-Public-IP-Address>
   ```

3. **Install MySQL server:**

   ```bash
   sudo yum install mysql-server -y
   sudo systemctl start mysqld
   sudo systemctl enable mysqld
   ```

4. **Secure MySQL installation:**

   ```bash
   sudo mysql_secure_installation
   ```

5. **Log in to MySQL:**

   ```bash
   mysql -u root -p
   ```

6. **Create the `tooling` database and user `webaccess`:**

   ```sql
   CREATE DATABASE tooling;
   CREATE USER 'webaccess'@'<Subnet-CIDR>' IDENTIFIED BY 'password';
   GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'<Subnet-CIDR>';
   FLUSH PRIVILEGES;
   ```

---

## **Step 3: Prepare the Web Servers**

1. **Launch 3 new EC2 instances with RHEL 8 for the web servers.**

2. **SSH into each web server instance:**

   ```bash
   ssh -i <Your-Private-Key.pem> ec2-user@<Web-Server-Public-IP-Address>
   ```

3. **Install NFS client and mount shared storage:**

   ```bash
   sudo yum install nfs-utils nfs4-acl-tools -y
   sudo mkdir -p /var/www
   sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
   ```

4. **Make NFS mount persistent after reboot:**

   ```bash
   sudo vi /etc/fstab
   ```

   Add the following line:

   ```bash
   <NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
   ```

5. **Verify the mount:**

   ```bash
   df -h
   ```

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

2. **Start and enable Apache:**

   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

---

## **Step 4: Deploy Tooling Application**

1. **Fork the tooling source code from the StegHub GitHub account to your GitHub account.**

2. **Clone the repository onto your web servers:**

   ```bash
   git clone <Your-GitHub-Repo-URL>
   ```

3. **Deploy the website to `/var/www/html`:**

   ```bash
   sudo cp -r /path/to/repo/html/* /var/www/html/
   ```

4. **Update the `functions.php` file to connect to the MySQL database by specifying the correct database credentials.**

5. **Apply the `tooling-db.sql` script to the database:**

   ```bash
   mysql -h <Database-Private-IP> -u webaccess -p tooling < tooling-db.sql
   ```

6. **Create a new admin user in MySQL:**

   ```sql
   INSERT INTO 'users' ('id', 'username', 'password', 'email', 'user_type', 'status') VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
   ```

---

## **Verification**

1. **Access the website

** via the web servers’ public IP address:

   ```bash
   http://<Web-Server-Public-IP>/index.php
   ```

You should see the tooling website. Confirm that you can log in with the admin credentials created in Step 4.

---

This completes the **DevOps Tooling Website Solution - 101** setup with **3 web servers**, NFS, and a remote MySQL database. You can now begin monitoring and improving this infrastructure.

---

This document covers all the details, including the SSH process, configuring 3 web servers, and minor but important steps for clarity.