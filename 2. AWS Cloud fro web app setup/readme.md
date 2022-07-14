## About the project
- Multi tier web application stac[VPROFILE]
- Host & Run on AWS Cloud for production
- Lift and shitt strategy

## Scenario
- App services are running on physical/virtual machine.
- work load in your data center
- Vitualization team, Ops, sysadmin, monitoring team etc. involved

## Problem
- Complex management
- scale up/down complexity
- upfront capEx & Regular OpEx
- manual process
- diff to automate
- time consuming

## Solution:
- Cloud computing setup
- PayAsGo, Flexibility,IAAS,etc

### AWS Services:
- EC2 Instances (VM for tomcat,rabbitMQ,memcache,mysql)
- ELB : loadbalancer (NGINX LB replacement)
- Autoscaling (Automation for VM Scaling)
- S3 / EFS Storage (Shared storage)
- Route 53 (Prvate DNS Service)

### Objectives:
- Flixible infra
- no upfront cost
- modernize effectively
- IAAC

## Flow of execution
### 1. Login to AWS Account
### 2. Create Key Pairs
- Go To EC2 service,Click on `Key Pairs`.
- Give a name and select `.pem` and create key pair. 

### 3. Create security groups
- vprofile-ELB-SG
    - This is ELB , that accepts the request from the users and redirect traffic to the out application servers.
    - it needs both HTTP & HTTPS ports to be opened

- vprofile-app-sg
    - It accepts traffic flow from ELB at TCP port 8080 and add `vprofile-ELB-SG` in the source
    - to SSH into this instance, open SSH port

- vprofile-backend-SG	
    - Here, backend contains memcache, rabbit mq, sql server
    - open the ports of these service and mention `vprofile-app-sg` in the source
    - Also allow all traffice to flow internal to this instance. so, mention `vprofile-backend-SG` in source.


### 4. Launch instances with user data [BASH SCRIPTS]
- These setup includes 3 instances
    1. Tomcat server (app01)
        - Create `Ubuntu` EC2 instance with security group as `vprofile-app-sg`
        - In the user data include below script:
        ```
            #!/bin/bash
            sudo apt update
            sudo apt upgrade -y
            sudo apt install openjdk-8-jdk -y
            sudo apt install tomcat8 tomcat8-admin tomcat8-docs tomcat8-common git -y
        ``` 
    2. RabbitMQ server (rmq01)
        - Create `CentOS` EC2 instance with security group as `vprofile-backend-SG`
        - In the user data include below script:
        ```
            #!/bin/bash
            sudo yum install epel-release -y
            sudo yum update -y
            sudo yum install wget -y
            cd /tmp/
            wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
            sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
            sudo yum -y install erlang socat
            curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
            sudo yum install rabbitmq-server -y
            sudo systemctl start rabbitmq-server
            sudo systemctl enable rabbitmq-server
            sudo systemctl status rabbitmq-server
            sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
            sudo rabbitmqctl add_user test test
            sudo rabbitmqctl set_user_tags test administrator
            sudo systemctl restart rabbitmq-server
        ``` 
    3. Database server (db01)
        - Create `CentOS` EC2 instance with security group as `vprofile-backend-SG`
        - In the user data include below script:
        ```
            #!/bin/bash
            DATABASE_PASS='admin123'
            sudo yum update -y
            sudo yum install epel-release -y
            sudo yum install git zip unzip -y
            sudo yum install mariadb-server -y

            # starting & enabling mariadb-server
            sudo systemctl start mariadb
            sudo systemctl enable mariadb
            cd /tmp/
            git clone -b vp-rem https://github.com/devopshydclub/vprofile-repo.git
            #restore the dump file for the application
            sudo mysqladmin -u root password "$DATABASE_PASS"
            sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
            sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
            sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
            sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
            sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
            sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
            sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
            sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
            sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-repo/src/main/resources/db_backup.sql
            sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

            # Restart mariadb-server
            sudo systemctl restart mariadb

            #starting the firewall and allowing the mariadb to access from port no. 3306
            sudo systemctl start firewalld
            sudo systemctl enable firewalld
            sudo firewall-cmd --get-active-zones
            sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
            sudo firewall-cmd --reload
            sudo systemctl restart mariadb

        ``` 
    4. Memcache server (mc01)
        - Create `CentOS` EC2 instance with security group as `vprofile-backend-SG`
        - In the user data include below script:
        ```
            #!/bin/bash
            sudo yum install epel-release -y
            sudo yum install memcached -y
            sudo systemctl start memcached
            sudo systemctl enable memcached
            sudo systemctl status memcached
            sudo memcached -p 11211 -U 11111 -u memcached -d  
        ``` 
### 5. Update IP to name mapping in route 53.
It is a DNS resolution service where we can map the IPs to the names. and refer the names in the properties instead of using IPs in our application.
    
- Create Hosted Region under `Route53`
    - Create Hosted Region
    - give name `vprofile.in` and enter type as `private`
    - give the region same as your ec2 instance & default VPC
- Create Record
    - give the record name for eg. `db01` for mysql , eg. `db01.vprofile.in`
    - give th eip address of the `db01` server
    - Create same for `rmq01` and `mc01` server 

### 6. Build application from source code
- Locally clone the app repo
- GoTo /repo-name/src/main/resources
- edit the `application.properties` file and change the backend endpoints as mentioned above step .
- come back to app directory where pom.xml file present and run below command
    ```
    mvn install
    ```
- Now it will create a target folder and war file

### 7. Upload to s3 bucket
- now install `awscli` in the server using `apt install awscli` command locally.
- To configure the aws cli , we will have to create the user in IAM service.
    -  create User with the policy permission `AmazonS3FullAccess`
    - download the key ID and Access key
- enter the details below in server
    ```
    aws configure
    Access ID: ****
    Access key: ***
    region: <region name>
    output type: JSON
    ```
- create a bucket in the aws
    ```
    aws s3 mb s3://<bucket name>
    ```
- copy the war file to the bucket
    ```
    aws s3 vprofile.war s3://<bucket name>
    ```
### 8. Download artifact to tomcat EC2 Instance

- Now we will have to create a Role for ec2 istance to get the war file from s3 bucket
- Under IAM, create Role , enter the AWS Service & EC2, then under permisson give S3 full access and role name.
- Now, modify the role under app01 instance.
    - actions -> instance settings -> modify the IAM Role and attach the above created role .
- Now ssh into `app01` server
    - go to /var/lib/tomcat8
    - stop tomcat service
    - delete default `ROOT` folder
    - now install aws cli 
    - copy the war file from s3 bucket
    ```
    aws s3 cp s3://<bucket-name> .
    ```
    - change the name to ROOT.war
    ```
    mv vprofile.war ROOT.war
    ```
    - restart tomcat server
    ```
    systemctl restart tomcat8
    ```

### 9. Setup ELB with HTTPS [Cert from Amazon Certificate Manager]
- Before creating ELB, we will have to create `Target Group` : Your load balancer routes requests to the targets in a target group and performs health checks on the targets.
- Click on create Traget Group, add new name and give port & path where the health check happens
- attach instance where app is hosted.
- Now create `Load Balancer`, select Application Load Balancer 
- Under security groups, attach `vprofile-ELB-SG` (It accepts the request from HTTP/HTTPS port). 
- in listener & routing session-> give port number and give target group name (This will check for health status)
- Provide certificate from AWS certificate manager which was created for `.*<domain>`

### 10. Map ELB Endpoint to website name in godaddy DNS
- Now DNS name will be displayed, copy it and go to domain provider, here it is godaddy and under domain management , specify CNAME . 
- Add name as project name eg, `vprofileapp` and value as DNS name which you copied.

### 11. Verify
- it should show active under ELB from provision state

### 12. Build Autoscaling Group for Tomcat Instances
- Before creating ASG, we will have to create VMI image
- under isntances,select ec2 app instance , Actions-> create Image
- give a name and create image and this will create `AMI` for us.
- Go to launch configuration for ASG
- Give a name, and select AMI which we created.
- instance type , give as what needed
- IAM Role, give a Role which we created for AWS S3 full access
- under security group, attach `vprofile-app-SG`. and click on create
- now create ASG, give a name, give a launch config template name you created before 
- Give a target group, if you enabled load balancing
- select min, max, desired capacity.
- You can add scaling policies if needed
- Add notification if instances is launched or terminated.
