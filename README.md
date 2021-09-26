# How to create various AWS resources using Terraform?

### What is Terraform?

Terraform is an open-source infrastructure as a code (IAC) tool that allows to create, manage & deploy the production-ready environment. Terraform codifies cloud APIs into declarative configuration files. Terraform can manage both existing service providers and custom in-house solutions.

![1](https://github.com/DhruvinSoni30/Terraform-AWS/blob/main/1.png)

### Prerequisite:
* Basic understanding of AWS & Terraform
* A server with Terraform pre-installed
* An access key & secret key created the AWS
* The SSH key

In this tutorial, I will be going to create various resources like VPC, EC2, SG, etc using terraform. So, let’s begin the fun.

> We will use separate variables file for storing all the variables. So, at the end I will discuss that file also.


**Step 1:- Create Provider block**
  
  ```
  provider "aws" {
    access_key = "${var.access_key}"
    secret_key = "${var.secret_key}"
    region     = "${var.aws_region}"
  }
  ```
  
* Provider block is used to configure and download plugins for the various cloud provider
* We are using variables for access_key, secret_key & region

**Step 2:- Creating AWS VPC**

  ```
  resource "aws_vpc" "demovpc" {
    cidr_block       = "${var.vpc_cidr}"
    instance_tenancy = "default"
  tags = {
    Name = "Demo VPC"
  }
  }
  ```
  
* aws_vpc is the VPC module for AWS
* We are using variable for cidr_block
* demovpc is the logical name of VPC resource
* I have set instance_tenancy to default
* Demo VPC is the tag of the VPC

**Step 3:- Creating Internet Gateway**

  ```
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }
  ```
  
* aws_internet_gateway is the Internet Gateway module for AWS
* demogateway is the logical name of Internet Gateway
* We are using the newly created VPC’s ID and attaching IGW to that VPC in this code vpc_id = “${aws_vpc.demovpc.id}"

**Step 4:- Updating the AWS Route Table**

  ```
  resource "aws_route" "internet_access" {
    route_table_id         = "${aws_vpc.demovpc.main_route_table_id}"
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = "${aws_internet_gateway.demogateway.id}"
  }
  ```

* aws_route is the Route Table module for AWS
* internet_access is the logical name of the Route table
* We are using the ID of the main route table in this block aws_vpc.demovpc.main_route_table_id
* We are adding the destination CIDR block as 0.0.0.0/0 for internet access.
* We are attaching the updated route table to the newly created IGW in this block aws_internet_gateway.demogateway.id

**Step 5:- Creating AWS Subnet**

  ```
  resource "aws_subnet" "demosubnet" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet_cidr}"
    map_public_ip_on_launch = true
  tags = {
    Name = "Demo subnet"
  }
  }
  ```

* aws_subnet is the subnet module for AWS
* demosubnet is the logical name of the subnet
* We are attaching the subnet to the newly created VPC in this block aws_vpc.demovpc.id
* We are using variable for cidr_block
* map_public_ip_on_launch will attach public IP to EC2 instances that are going to launch in this subnet
* Demo subnet is the tag

**Step 6:- Creating Inbound AWS Security Group**

  ```
  resource "aws_security_group" "demosg" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  # Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ```
  
* aws_security_group is the Security Group module for AWS
* demosg is the logical name of the subnet
* We are creating this SG in the newly created VPC in this code aws_vpc.demovpc.id
* We are creating Inbound rules for 80, 443 & 22 ports and allowing access for all the IPs

**Step 7:- Creating Outbound AWS Security Group**
  
  ```
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```

* We are opening outbound connection for all the IPs from anywhere

**Step 8:- Creating key pair for AWS EC2 Instance**

  ```
  resource "aws_key_pair" "demokey" {
    key_name   = "${var.key_name}"
    public_key = "${file(var.public_key)}"
  }
  ```
  
* aws_key_pair is the key pair module for AWS
* demokey is the logical name for key pair
* We are using the file function to take the value from the public_key variable for the key pair
* So, basically, it will take the value of the SSH key pair from the tests.pub file.

![2](https://github.com/DhruvinSoni30/Terraform-AWS/blob/main/2.png)

**Step 9:- Creating AWS EC2 instance**

  ```
  resource "aws_instance" "master" {
  ami = "${var.ami}"
  subnet_id = "${aws_subnet.demosubnet.id}"
  instance_type = "${var.instancetype}"
  
  count= "${var.master_count}"
  
  key_name = "${aws_key_pair.demokey.id}"
  vpc_security_group_ids = ["${aws_security_group.demosg.id}"]
  tags = {
    Name = "Cluster-Master"
  }
  root_block_device {
    volume_size = "40"
    volume_type = "standard"
  }
  
  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = "80"
    volume_type = "standard"
    delete_on_termination = false
  }
  ```
* aws_instance is the EC2 instance module for AWS
* master is the logical name of EC2 instance
* For AMI ID we are using the value from the variable
* We are creating the EC2 instance in the newly created subnet in this code aws_subnet.demosubnet.id
* We are using the value from the variable for count
* We are attaching the newly created key pair for SSH access in this code aws_key_pair.demokey.id
* We are attaching the newly created SG to the EC2 instance in this code aws_security_group.demosg.id
* We are extending the side of Root volume to 40 GB
* We are also creating the 80 GB additional volume and attaching it to /dev/sdb which of standard type and it will not delete while terminating the instance

**Step 10:- Creating Connection block**
 
  ```
  connection {
   
    user = "ec2-user"
    private_key = "${file(var.private_key)}"
    type = "ssh"
  }
  ```

* For configuring the EC2 instance we need to use the connection block and need to define how we are going to connect to our EC2 instance
* In this case, we are using SSH connection
* We are using the file function to take the value from the private_key variable for the key pair
* So, basically, it will take the value of the SSH key pair from tests.pem file so, that we can SSH into the EC2 instance using that key

![2](https://github.com/DhruvinSoni30/Terraform-AWS/blob/main/2.png)

**Step 11:- Deploying Apache Webserver**

* Now our infrastructure is up to date and it is ready for configuration. So, now we are using the user data block to configure it and install the Apache Webserver
* We using the shell script to install the apache webserver
  ```
  #! bin/bash
  yum install httpd -y
  service httpd start
  chkconfig httpd on
  cd /var/www/html
  touch index.html
  echo "<h1> Dhruvin Soni </h1> > /var/www/html/index.html
  ```
  
* And here is our user data block.

  ```
  user_data = "${file("data.sh")}"
  ```
  
* Here is my variable file:

  ```
  # Defining Access Key
  variable "access_key" {}
  
  # Defining Secret Key
  variable "secret_key" {}

  # Defining Public Key
  variable "public_key" {
    default = "tests.pub"
  }
  
  # Defining Private Key
  variable "private_key" {
    default = "tests.pem"
  }
  
  # Definign Key Name for connection
  variable "key_name" {
    default = "tests"
    description = "Desired name of AWS key pair"
  }

  # Defining Region
  variable "aws_region" {
    defaultt = "us-east-1"
  }
  
  # Defining CIDR Block for VPC
  variable "vpc_cidr" {
    default = "10.0.0.0/16"
  }

  # Defining CIDR Block for Subnet
  variable "subnet_cidr" {
    default = "10.0.1.0/24"
  }
  
  # Defining AMI
  variable "ami" {
    default = {
    us-east-1 = "ami-09d95fab7fff3776c"
  }
  }
  
  # Defining Instace Type
  variable "instancetype" {
    default = "t2.medium"
  }

  # Defining Instance count 
  variable "master_count" {
    default = 1 
  }
  ```
  
* So, now our entire code is ready. We need to run the below steps to create infrastructure.
* terraform init is to initialize the working directory and downloading plugins of the provider
* terraform plan is to create the execution plan for our code
* terraform apply is to create the actual infrastructure. It will ask you to provide the Access Key and Secret Key in order to create the infrastructure. So, instead of hardcoding the Access Key and Secret Key, it is better to apply at the run time.

After the infrastructure is ready you can verify the apache webserver’s output by navigating http://ec2-ip you should see the default apache webserver's output.

That’s it now, you have learned how to create various resources in AWS using Terraform. 
