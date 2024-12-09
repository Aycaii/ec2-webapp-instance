# ec2-webapp-instance
A step by step process of my web application built on an AWS Academy environment


For this project design I have implemented 
![image](https://github.com/user-attachments/assets/7d30e960-af33-49c3-ba14-1990d2a9ef8a)

Phase 2: Creating a basic functional web application
Read the instructions in the Vocareum lab. The steps below will guide you through exactly how to carry out the tasks. Note that details can be very important, so try to follow the instructions as closely as possible. If at any point what you're seeing is different from what is indicated in these instructions and screenshots, ask for some help.

Task 1: Creating a virtual network
1. Launch the Vocareum project environment and log into AWS by clicking "AWS" in the upper left corner of the screen. 
2. In AWS, search for VPC in the search bar, and navigate to the VPC page:002_to_vpc.png
3. Select the default VPC by clicking the check box to the left of the VPC in the list. From "Actions" choose "Delete VPC"
4. Confirm the deletion by typing "delete default vpc" in the field when prompted
5. Click the "Create VPC" button
6. Choose "VPC and more". This will create a default network structure including four subnets, a public subnet and a private subnet across two availability zones. 7. Confirm that the structure shown in the Preview area corresponds with this <img width="1284" alt="003a" src="https://github.com/user-attachments/assets/68817eb3-282d-40c0-b4f5-a034c1ff1816">

7. Under VPC settings, take a look at the values. Leave all of them as the default values. This should give us a new VPC with 2 public subnets and 2 private subnets, which is perfect for this project. The name tag is automatically set as "project" which is also a good name for tags for this project. Click "Create VPC" at the bottom of the page and wait while the VPC gets set up. When the bullet points in the creation process all display green checkmarks as shown below click "View VPC".


Task 2: Creating a virtual machine
We'll continue with launching a single virtual machine (EC2 instance) to run our application as a proof of concept.

Since this machine will be running a web server, which must be accessible from the Internet, the machine will need to be deployed in a public subnet. We'll opt to put it in Public Subnet 1.

Navigate to the EC2 Dasbhoard in the AWS Console. You can find this by using the search field, as you did with the VPC Dashboard.Click the orange launch instance button shown here:
Click the orange "Launch instance" button shown here:004_launch_inst.png
Set the following values (leaving other values at their default):
Name: Web Server with Database
AMI: Ubuntu Server (click the "Ubuntu" card under "Quick Start")
Key pair name: vockey (select from drop-down menu)
To the right of the Network settings header, click Edit and enter the network settings area.
In the "VPC" drop-down menu, select the VPC labeled project-vpc.
In the "Subnet" drop-down menu, select the subnet labeled project-subnet-public1.
In the "Auto-assign public IP" drop-down menu, select Enable.
Click "Add security group rule"
For the new security group rule,
In the "Type" drop-down menu select HTTP
In the "Source type" field, select Anywhere
Open "Advanced details"
In the "User data - optional" text field, copy and paste the following shell script, which will be executed when the instance is launched. This will install all the necessary software for our application. 

#!/bin/bash -xe
apt update -y
apt install nodejs unzip wget npm mysql-server -y
#wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-DEV/code.zip -P /home/ubuntu
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-79581/1-lab-capstone-project-1/code.zip -P /home/ubuntu
cd /home/ubuntu
unzip code.zip -x "resources/codebase_partner/node_modules/*"
cd resources/codebase_partner
npm install aws aws-sdk
mysql -u root -e "CREATE USER 'nodeapp' IDENTIFIED WITH mysql_native_password BY 'student12'";
mysql -u root -e "GRANT all privileges on *.* to 'nodeapp'@'%';"
mysql -u root -e "CREATE DATABASE STUDENTS;"
mysql -u root -e "USE STUDENTS; CREATE TABLE students(
            id INT NOT NULL AUTO_INCREMENT,
            name VARCHAR(255) NOT NULL,
            address VARCHAR(255) NOT NULL,
            city VARCHAR(255) NOT NULL,
            state VARCHAR(255) NOT NULL,
            email VARCHAR(255) NOT NULL,
            phone VARCHAR(100) NOT NULL,
            PRIMARY KEY ( id ));"
sed -i 's/.*bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl enable mysql
service mysql restart
export APP_DB_HOST=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
export APP_DB_USER=nodeapp
export APP_DB_PASSWORD=student12
export APP_DB_NAME=STUDENTS
export APP_PORT=80
npm start &
echo '#!/bin/bash -xe
cd /home/ubuntu/resources/codebase_partner
export APP_PORT=80
npm start' > /etc/rc.local
chmod +x /etc/rc.local


Although it's not necessary to understand all of this, it's worthwhile to give it a look over. Mostly these are simply Unix command line commands (you may remember apt from Lab 1 when you installed software on Ubuntu). The web application code is being downloaded on line 5 with the wget command. The application has been written in advance and is provided by AWS at the URL shown. Also note that the database is being created and some details about its username and password are set.
Click "Launch Instance".
You can view your instances by going to the "Instances" panel (in the left side menu) of the EC2 Dashboard. You will see a list of instances that have been launched (there should only be one there at this point). Your instance will say "Pending" under "Instance state" for a while, and eventually (after a few minutes) change to "Running". The "Status checks" column should eventually read "2/2 checks passed".

Task 3: Testing the deployment
Once the instance is fully running with status checks passed, click on its instance ID number to view the details of the instance.

Find "Public IP address". You can either copy the value and paste it into a browser URL window, or you can click "open address" which will open this IP address directly into a browser window. You may see something like this:006_unable.png
The problem here is that by default, your browser sends requests to a web server using the "https" (encrypted) protocol. But we've only set our application up to allow plain "http" requests (unencripted web requests). All you need to do is to delete the "s" in the URL field of your browser, so that the URL begins with "http://" instead of "https://". Once you've done this, you should see the application

<img width="1125" alt="007_app_screen" src="https://github.com/user-attachments/assets/a5baf58f-66a3-4e64-a879-b272726ba381">



For reasons of scalability and fault tolerance, we want to decouple the database from the application instance. We'll create an AWS RDS database to handle our SQL database needs.

First, identify the IDs and IP address block values of the private subnets. To do this, navigate to to the VPC dashboard, click Subnets, and look at the list of subnets, which should look something like the image below:007d_note_subnet_info.png Make notes of the values corresponding to what I've highlighted (your values will be different from these). Be sure to note the values for the private subnets. Copy the values somewhere so that you can use them later.
Navigate to the RDS Dashboard (you can search for "RDS" to get the link quickly).
You'll need to create a subnet group to determine where the RDS instance will reside. Click the "Subnet groups" link in the left-and side menu on the RDS Dashboard.
Click the "Create DB subnet group" button.
In the "Name" field enter DB-Subnet-Group
In the "Description" field enter DB Subnet Group
From the VPC drop-down menu, choose the VPC with the name project-vpc
Under Add subnets, in the "Availability Zones" drop-down menu, check us-east1a and us-east1b, as shown:007f_add_subnets.png
When you've selected the availability zones, you will see a drop-down to select subnets, like this:1.png Select the two subnets with the name project-subnet-private.
Click "Create"
Navigate to "Databases" in the RDS Dashboard (left-hand side menu)
Click "Create Database"
Under "Database Engines" select MySQL
Under "Templates" choose Dev/Test
Under "Availability and Durability" select Multi-AZ DB Instance
Under "Settings", in "DB Instance identifier" put STUDENTS
Under "Credentials settings" choose Self Managed
Set the db username and password
For "Master username" enter nodeapp
For "Master password" enter student12
Note that these must be exactly accurate
Under "Instance configuration" select Burstable classes (includes t classes)
In the drop-down menu right below that select db.t3.micro
Under Storage, in the "Allocated storage" field, put 20
Under Connectivity, for the "Virtual private cloud (VPC)" drop-down menu, select the project-vpc VPC.
Under "VPC security group (firewall)" choose Create new
In the "New VPC security group name" field put db-security-group
Unselect the "Enable Enhanced Monitoring" check box
Open the "Additional configuration" panel
Under "Database options", in "Initial database name" enter STUDENTS
Uncheck "Enable automated backups"
Note the estimated monthly cost. You should be seeing estimates approximately equal to
DB instance 24.82 USD
Storage 4.69 USD
Total 29.42 USD
This gives you some idea of how this service will impact your budget. Remember, you have a $100 budget, and this RDS instance only needs to run for as long as it takes to complete the lab. So there should be plenty of money in the budget to cover this cost. If your total is significantly different from this, go back over your settings and make sure you've followed the instructions closely.
Click "Create database". You can simply close the panel that appears at this point recommending database add-ons
The database will be in "Creating" status for a while then go through a few status changes before landing on "Available". This may take some time so it's a good time to take a break and come back to it in 15 minutes or so.

Task 3: Configuring the development environment
It will help to do this in a separate browser tab, so that  you can keep your Cloud 9 developmen environment open while navigating around the AWS Console in a separate tab.

Navigate to the Cloud 9 Dashboard in the AWS Console (you can find it using the Search tool as before), and right click on the link to "Open in New Tab".
Click "Create Environment"
Give it the name "Dev Environment"
Under "Network settings" for "Connection" choose "Secure Shell (SSH)"
Under "VPC settings" for "Amazon Virtual Private Cloud" choose the project-vpc VPC. Under "Subnet" choose the public1 subnet.
Once your Cloud9 environment is created, open it by clicking "Open" in the Cloud9 IDE column. This will give you a view into an IDE that is very similar to VSC, with an overview of files on the left, a text editor space, and a Unix-style command line interface in the lower right. You will be using this command line environment to run the scripts described in the next several sections.

Task 4: Provisioning Secrets Manager
Keeping the Cloud9 tab open, in a separate tab navigate to your RDS Dashboard and copy the RDS Endpoint from your database's details panel, as shown here (Note: in the screenshot below, the database name shown is database-1 following the instructions above should result in your database called students. This is an inaccuracy in the screenshot, please disregard the different database name):008_endpoint.png
Run the script below to create a secret in the AWS Secrets Manager (This script is "Script 1" in the AWS Cloud9 Scripts downloadable in the Task 4 instructions). Replace the <RDS Endpoint> below with your actual RDS endpoint copied in the previous step.

aws secretsmanager create-secret \
    --name Mydbsecret \
    --description "Database secret for web app" \
    --secret-string "{\"user\":\"nodeapp\",\"password\":\"student12\",\"host\":\"<RDS Endpoint>\",\"db\":\"STUDENTS\"}"

Also, copy the endpoint somewhere you can get to it easily, because you're going to need it again later. The output of running the script should look something like this:

{
    "ARN": "arn:aws:secretsmanager:us-east-1:878416559186:secret:Mydbsecret3-kqbBKX",
    "Name": "Mydbsecret",
    "VersionId": "dff92fc1-b9e8-4889-a2e7-ee95bd937aa1"
}


Task 5: Provisioning a new instance for the web server
In the EC2 Dashboard, click "Launch an Instance"
Enter the following settings:
Name: Web Server Decoupled
AMI: Ubuntu
Instance type; t2.micro
Key pair: vockey
Click "Edit" on the Network settings tab.
In the Network settings, from the "VPC" drop-down menu choose the project-vpc VPC
In the "Subnet" drop-down menu, select project-subnet-public-1-useast-1a
For the security group, select "Existing security group" and choose the launch-wizard-1 security group (the same one used by the previously created web server). Make sure no other security groups are attached.
Under "Auto-assign public IP" select Enable
Open the Advanced details panel
In Advanced details, in the "IAM instance profile" drop-down menu, select LabInstanceProfile
Paste the following script into the "User data" text field:
#!/bin/bash -xe
apt update -y
apt install nodejs unzip wget npm mysql-client -y
#wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-DEV/code.zip -P /home/ubuntu
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-79581/1-lab-capstone-project-1/code.zip -P /home/ubuntu
cd /home/ubuntu
unzip code.zip -x "resources/codebase_partner/node_modules/*"
cd resources/codebase_partner
npm install aws aws-sdk
export APP_PORT=80
npm start &
echo '#!/bin/bash -xe
cd /home/ubuntu/resources/codebase_partner
export APP_PORT=80
npm start' > /etc/rc.local
chmod +x /etc/rc.local
  
Click "Launch Instance"
Task 6: Migrating the database
For this task, you're going to export the contents of your original database (the one running on the EC2 instance with the first web server) and then import those contents into the new database running on RDS. The commands to do this you will run from the Cloud 9 environment, but first it's necessary to ensure that the Cloud 9 environment has the necessary permission to communicate with the instance.

Navigate to EC2 Dashboard for the original "Web Server with Database" instance.
Click through to the detailed information dashboard for the instance. Go to the "Security" panel and click on the link to the security group it is in (launch-wizard-1)
Click "Edit Inbound Rules"
Click "Add rule"
In the "Type" drop-down menu, select MySQL/Aurora
Click on the Source field and from the drop-down menu select the security group associated with the Cloud 9 dev environment, as shown here:012a_edit_inbound_rules.png
Click "Save rules"
Return to the EC2 dashboard. Select the "Web Server with Database" instance (the first one you created) and copy its private IP address, as shown here:014_privateIPV4_address.png
Return to the Cloud 9 environment
Run the following command in the command line in Cloud 9, replacing <EC2instancePrivateip> with the private IP address of the instance you just copied above. (This command is taken from the cloud9 scripts linked to in Task 4 instructions):
mysqldump -h <EC2instancePrivateip> -u nodeapp -p --databases STUDENTS > data.sql
In my environment, running this command looks like this, but your IP address will be different: 015_command.png When you do this, you will be prompted for a password. The password is student12
The result should be a new file called "data.sql". You can see this by running the ls -al command on the command line which lists the contents of the directory, like this:016_ls-al.png
Next, we'll complete the migration by migrating the dumped output of the database into the new database using a similar script as before. This time, we will need to modify the RDS instance's security group to allow our requests. Navigate to the RDS Dashboard detail view of instance students (Note: in the screenshot below, the database name shown is database-1 following the instructions above should result in your database called students. This is an inaccuracy in the screenshot, please disregard the different database name). Find the link to the security group for the RDS instance under "Connectivity & Security":017_security_group.png
Click the security group link. Click the "Inbound rules" tab.
Add a new rule with type "MySQL/Aurora" and for the source select the security group associated with the Cloud 9 Dev Environment, so the result looks something like this:019_.png
Add another new rule, also with type "MySQL/Aurora" and for the source of this new rule select the security group associated with your Web Server Decoupled instance (the name of this security group will be launch-wizard-1). This is needed to ensure that your application can communicate with the database:Screenshot 2024-06-10 at 14.31.42.png
Click "Save rules"
To restore the old database contents to the new database you'll use another script from the cloud9-scripts.yml document provided in Task 4 instructions:
mysql -h <RDSEndpoint> -u nodeapp -p  STUDENTS < data.sql
Note that <RDSEndpoint> needs to be replaced by your RDS instance's endpoint value. Refer back to Task 4, where you made a note of the RDS endpoint.
Task 7: Testing the application
Return to your "Web Server Decoupled" EC2 Instance. Copy and paste its public IP address into your browser's URL field.

You should now be able to add and edit students in the application running on "Web Server Decoupled".

This was probably the most difficult and prone-to-error portion of the project. If it's working now, give yourself a pat on the back! The most likely issues to arise are security group access permissions and database username/password access issues. 

If you encountered issues, there are a couple of things to check. Did the SQL dump operation work properly? You may be able to tell what got exported by looking into the data.sql file, which is human readable. Does it seem to have references to the student names you input in your previous testing? If not, the export may have failed. If so, you should focus on the importing step. In either case, look closely at security groups and make sure the values you used in your command line commands are correct.

Phase 4: Implementing high availability and scalability
Your application is now decoupled in the sense that the web server instance no longer hosts the database. This is an important step. However, since there's only a single web server instance, the application still depends on that single instance. If the instance crashes, the whole application goes down. This is a design flaw known as a single point of failure and it's important to avoid it. We want to make sure that a single instance failure (which happens all the time) does not cause our application to fail.

You'll do this by implementing redundancy and scalability by using a load balancer to distribute the workload across multiple copies of the instance in an auto-scaling group that can add instances or terminate instances depending on the traffic. Let's look at how to do this.

Task 1: Creating an Application Load Balancer
Navigate to "Load Balancers" by finding it in the Search tool.
Click "Create Load Balancer"
Choose "Application Load Balancer". Click "Create".
Under "Basic configuration"
Under "Load Balancer Name" put project-load-balancer
Under "Network mapping"
In the VPC drop-down menu choose the project-vpc VPC
Under "Mappings", check both us-east-1a and us-east-1b
In the "Subnet" drop-down menus, choose the subnets with public in their
name (not the ones with private in their names!)
For "Security Group" select launch-wizard-1 and deselect default
Under "Listeners and routing" click the "create target group" link. This will open a new panel for you to create a target group for the load balancer
In the new "Create target group" tab, under "Basic configuration":
Target type: Instances
Target group name: web-server-target-group
VPC: project-vpc
Click "Next"
Under "Register Targets", in "Available Instances" check the box next to "Web Server Decoupled"
Click "Include as pending below"
Under "Review targets" double check that "Web Server Decoupled" is listed as a target. Then click "Create target group"
Return to the browser tab where you are setting up the load balancer. Click the circular-arrow refresh key to the right of the target groups drop-down to make sure that the new target group shows up there, then select it in the drop-down menu.
Click "Create load balancer"
When the load balancer turns to "Active" find the DNS name for the load balancer on the load balancer's "Details" panel. Copy the DNS name into your browser URL field. You should see your application and be able to add and edit student data.
Congratulations! You've now deployed your application behind a load balancer.

Task 2: Implementing Amazon EC2 Auto Scaling
An auto-scaling group is composed of multiple copies of an instance which can be scaled out by adding more copies of the instance or scaled in by terminating copies. In order to create these copies, we need to create an AMI of our functioning instance. We'll create this AMI from the "Web Server Decoupled" instance.

Navigate to the EC2 Dashboard, and select the "Web Server Decoupled" instance.
In the "Actions" menu, select "Images and templates > Create Image"
In the "Image name" field enter web-server-decoupled-image
In the "Image description" field enter Web Server Decoupled Image
Click "Create Image"
In the "AMIs" page in the EC2 Dashboard select the new image from the "Images" list and view the details. Wait until the AMI's status reads "Available"
Navigate to the "Auto Scaling Groups" page in the EC2 Dashboard
Click "Create Auto Scaling Group"
In the "Name" field enter: web-server-auto-scaling-group
Click "Create a launch template". This will open a separate browser tab.
In the "Create launch template" tab enter
"Launch template name": web-server-launch-template
Under "Application and OS Images" choose "My AMIs". The web-server-decoupled-image AMI should appear in the Amazon Machine Image drop down
For "Instance type" choose t2.micro
For "Key pair (login)" choose vockey
For "Security groups" choose the launch-wizard-1 security group
Open the "Advanced details" panel
Under IAM instance profile, select LabInstanceProfile
Click "Create Launch Template"
Return to the Create Auto Scaling group browser tab
Click the circular arrow refresh button to the right of the launch templates drop-down menu to ensure that your newly created launch template is listed, then select web-server-launch-template from the drop-down menu
Click "Next"
In "Choose instance launch options" under "Network" make sure that project-vpc is selected.
For "Availability and subnets", check both public subnets listed in the drop-down. Keep the private subnets un-checked, as shown:020_network.png
Click "Next"
Under "Configure advanced options"
For "Load balancing" choose Connect to an existing load balancer
Under "Attach to an existing load balancer" select Choose from your load balancer target groups
Choose the web-server-target-group from the drop-down menu
Under "Health checks" check Turn on Elastic Load Balancing health checks
Click "Next"
On the "Configure group size and scaling" page set the following values:
In "Desired capacity": 2
Under "Scaling" 
Min desired capacity: 2
Max desired capacity: 3
for "Automatic scaling" choose Target tracking scaling policy
Click "Next"
On the "Optional notifications" page, click "Next"
On the "Optional tags" page, click "Next"
Review and click "Create Auto Scaling group"
Your load balancer should now begin to send traffic to the auto scaling group.

Task 3: Accessing the application
Take a look at the list of your instances on the EC2 dashboard. To test that the auto scaled instances are running the application correctly, make sure they are fully in Running status with their status checks passed, and then stop the original "Web Server Decopoupled" instance. Do this by checking the selection box next to "Web Server Decoupled", and from the "Instance State" drop-down menu select "Stop Instance" (don't terminate the instance, just in case you need it later for some troubleshooting).

If you still have the application in your browser (with the load balancer's DNS name as the URL) then refresh your browser to make sure the application is still working. If not, then once again copy your load balancer's DNS name (available in the load balancer details page) into the URL field of your browser to test the application. 

Task 4: Load testing the application
Now you'll run a script to test the application's capacity. These basic commands are also in the downloaded scripts, but I've set some values in a way that might be useful, so you should just enter in what's written below.

Install the loadtest utility with the command:
npm install -g loadtest
Now run the loadtest utility by typing the command below into your Cloud9 command line (replace <Your load balancer URL> with your actual load balancer URL, which you used to view the application. Your load balancer URL will look something but not exactly like http://project-load-balancer-1319799661.us-east-1.elb.amazonaws.com. Be sure to copy the URL that does not end with 'students'. If we do the load test on the page that makes the database calls we may overwhelm the database, which is a separate thing from what we are testing here.
loadtest -t 300 -c 150 -k <Your load balancer URL>
This will bombard your application with requests, simulating a situation hundreds or thousands of viewers are looking at your application. I've set the concurrency value at 150, meaning that up to 150 requests can go out before the load test utility receives a response. This is just low enough that the server can keep up with most of the requests. You should expect to see a few errors (dropped requests), but not too many.

This will also run the load test for 300 seconds, which is as long as your auto scaling needs to determine that there's a heavy load on the two instances in the auto scaling group. When the loadtest finishes, navigate to your auto scaling group page on the EC2 dashboard. You should see that the "Desired capacity" value is now 3, rather than 2. You should also see that there are three instances running in your EC2 dashboard instances list.
