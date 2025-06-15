# 🎓 Student Management Web Application – AWS 3-Tier VPC Deployment Guide

## 📌 Overview

This project is a Java-based **Student Management System** deployed on **AWS** using a secure and scalable **3-tier VPC architecture**. It allows you to manage student records — add, edit, delete, and list — through a web interface.

---

## 🧱 3-Tier Architecture Overview

This deployment follows a standard **3-tier model**:

1. **Public Layer**: Bastion(proxy) Host (SSH access only)
2. **Application Layer**: EC2 instance running Apache Tomcat and Java WAR file
3. **Database Layer**: MySQL RDS instance in a private subnet

---

## 🛠️ Infrastructure Setup – Step-by-Step

### ✅ Step 1: Create VPC

- **Name**: `3-tier-vpc`
- **CIDR Block**: `10.0.0.0/16`
- **Tenancy**: Default

![VPC Setup](vpc.png)

---

### ✅ Step 2: Create 3 Subnets

| Subnet Name     | Type     | CIDR Block   | Purpose                |
|------------------|----------|---------------|--------------------------|
| `public-subnet`  | Public   | `10.0.1.0/24` | Bastion Host (SSH)       |
| `pvt-app-subnet`     | Private  | `10.0.2.0/24` | Tomcat Server (EC2)      |
| `pvt-db-subnet`      | Private  | `10.0.3.0/24` | MySQL RDS (Database)     |

![Subnet Setup](subnet.png)

---

### ✅ Step 3: Create Internet Gateway (IGW)

- **Name**: `3-tier-igw`
- Attach to: `3-tier-vpc`

![IGW Setup](create%20igw.png)

---

### ✅ Step 4: Create NAT Gateway

- Launch a **NAT Gateway** in the `public-subnet`
- Allocate an **Elastic IP**

![NAT Setup](create%20NAT.png)

---

### ✅ Step 5: Route Tables

#### 🔹 Public Route Table

- **Name**: `public-rt`
- Add route: `0.0.0.0/0 → IGW`
- Associate with: `public-subnet`

!(![pub-RT](<add igw to pub-RT.png>)

#### 🔹 Private Route Table

- **Name**: `private-rt`
- Add route: `0.0.0.0/0 → NAT Gateway`

![pvt-RT](<add NAT to pvt-RT.png>)

- Associate with: `app-subnet`, `db-subnet`

![sub_assocition](<subnet association.png>)
---

## 🚀 Application Deployment

---

### ✅ Step 6: Launch EC2 Instances

1. **Launch three EC2 instances in the respective subnets:**

- **Proxy Server** (Public Subnet): Install and configure NGINX as a reverse proxy
- **App Server** (Private Subnet): Deploy Apache Tomcat and the `student.war` file
- **DB Server** (Private Subnet): Install MySQL to manage student records

Assign appropriate security groups to allow:
- SSH from Bastion or trusted IPs
- HTTP traffic from Proxy to App Server (port 8080)
- MySQL access from App Server only (port 3306)

![instances](<launch 3 instances.png>)
---

2. **Install Java & Tomcat:**
   ```bash
   sudo yum update -y
   sudo yum install java-1.8.0-openjdk -y
   cd /opt
   curl -O https://downloads.apache.org/tomcat/tomcat-9/v9.0.86/bin/apache-tomcat-9.0.86.tar.gz
   sudo tar -xvzf apache-tomcat-9.0.86.tar.gz
   sudo mv apache-tomcat-9.0.86 tomcat

3. **Deploy student.war to Tomcat:**
      ```bash
      sudo cp student.war /opt/tomcat/webapps/

4. **Deploy student.war to Tomcat:**
      ```bash 
      cd /opt/tomcat/bin
      ./catalina.sh start

---

### ✅ Step 7: Add MySQL JDBC Connector

1. **Download the connector:**
      ```bash 
      cd /opt/tomcat/lib
      curl -O https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-8.0.33.tar.gz
      tar -xvzf mysql-connector-java-8.0.33.tar.gz
      sudo cp mysql-connector-java-8.0.33/mysql-connector-java-8.0.33.jar .


2. **Ensure it's mysql.connector.jar available in /opt/tomcat/lib for JDBC to work:**
---

### ✅ Step 8: Create RDS (MySQL) in Private Subnet

1. **Create RDS (MySQL) in Private Subnet**

   -  Engine: MySQL

   -  Version: 8.x

   -  DB Name: rdsdb

   -  Master Username: admin

   -  Password: yourpassword

   -  VPC: 3-tier-vpc

   -  Subnet Group: Select db-subnet

   -  Public Access: NO

   -  Security Group: select rds-sg

   ---

### ✅ Step 9: Configure Application with RDS

Configure the Java application to connect to the RDS or DB Server by adding the JDBC connection in the Tomcat `context.xml` file.

#### 🛠️ Update `context.xml`

Edit the file at:  
`/opt/tomcat/conf/context.xml`

Add the following inside the `<Context>` tag:

Resource 
         
          name="jdbc/studentdb" 
          auth="Container"
          type="javax.sql.DataSource"
          maxTotal="100"
          maxIdle="20"
          maxWaitMillis="-1"
          username="admin"
          password="yourpassword"
          driverClassName="com.mysql.cj.jdbc.Driver"
          url="jdbc:mysql://your-rds-endpoint:3306/rdsdb"

- Replace <rds-endpoint> with the actual RDS endpoint or the private IP of your MySQL EC2 instance
- Ensure the username and password match your DB credentials.

🔁 Restart Tomcat Server

    cd /opt/tomcat/bin 
    ./catalina.sh start
   
### ✅ Step 10: Set Up Reverse Proxy with NGINX

The Proxy Server (public subnet) uses NGINX to forward HTTP requests to the App Server (private subnet) where the Tomcat application is running.

#### ⚙️ Install NGINX on Proxy Server
      sudo (yum) install nginx1 -y
      sudo systemctl enable nginx
      sudo systemctl start nginx

- 📄 Configure nginx.conf

      sudo nano /etc/nginx/nginx.conf

- Inside the server block add like this:
      server {
       listen 80;

      location / {
        proxy_pass http://<app-server-private-ip>:8080/student/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
      }

- 🔁 Replace <app-server-private-ip> with the private IP of your App Server EC2 instance.
- Restart nginx service

### ✅ Step 11: Verify and Test the Full 3-Tier Application

After setting up the Proxy, App, and DB servers with all configurations, perform end-to-end testing.

#### 🌐 Access the Application

Open a web browser and enter the public IP of the Proxy Server:

![student_form](<stud reg form.png>)

This request will go through:
- NGINX on the Proxy Server (Public Subnet)
- → to Tomcat on the App Server (Private Subnet)
- → which connects to the MySQL database on the DB Server (Private Subnet or RDS)

![mysql_entry](<student table.png>)

#### 🧪 Test Functionalities

Use the UI to test these actions:

- ✅ Add new student records
- ✅ Edit existing student details
- ✅ Delete students
- ✅ View all student records

#### 📋 Additional Verifications

- NGINX is successfully forwarding traffic to Tomcat
- Tomcat is running and serving the `student.war` file
- MySQL connector is working and DB connection is active
- Database entries are being updated properly

#### 📁 Logs (Optional)

To debug issues, check:
- **NGINX logs**: `/var/log/nginx/access.log` and `error.log`
- **Tomcat logs**: `/opt/tomcat/logs/`
- **MySQL logs** (if on EC2): `/var/log/mysqld.log`

---

✅ With this, your Student Management Web App is fully deployed on a secure AWS 3-tier architecture.
---

### 🙋‍♂️ Author

This project setup and deployment was completed by **Kishor Borse**.
