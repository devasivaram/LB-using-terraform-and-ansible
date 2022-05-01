# LB-using-terraform-and-ansible

## Description

Creating an Classic Load Balancer(CLB) using terraform and provisioning using ansible, here we create security group, CLB and its listner, also the Lauch configuration and auto scaling group as well. While performing provisioning we use serial flag so that instance won't get deleted or recreated but changes will get applied to all instance.

## Prerequisites
-------------------------------------------------- 

Before we get started you are going to need so basics:

* [Basic knowledge of Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
* [Terraform installed](https://www.terraform.io/downloads)
* [Valid AWS IAM user credentials with required access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
* Install Ansible:
  ~~~
  sudo amazon-linux-extras install python3.8
  sudo pip3.8  install ansible
  sudo pip3.8  install boto3 boto botocore
  ~~~

## Installation

If you need to download terraform , then click here [Terraform](https://www.terraform.io/downloads)

First create a directory for our project (myproject), Lets create a file for declaring the variables.This is used to declare the variables that pass values through the terrafrom.tfvars file.

~~~
mkdir myproject
cd myproject
~~~

## Create a varriable.tf file

~~~
variable "region" {
  default = "ap-south-1"
}

variable "access_key" {
  description = "my access key"
  default = "add ur key"
}

variable "secret_key" {
  description = "my secret key"
  default = "add ur key"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "instance_ami" {
  default = "ami-0a3277ffce9146b74"
}

variable "project" {
  default = "new"
}
~~~

## Create a provider.tf file

~~~
provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
}
~~~

## Create a setup.sh

~~~
#!/bin/bash

echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config
echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment

yum install httpd php git -y
git clone https://github.com/devasivaram/aws-elb-site.git /var/website/
cp -r /var/website/* /var/www/html/
chown -R apache:apache /var/www/html/*
systemctl restart httpd.service
systemctl enable httpd.service
~~~

## Create outout.tf

~~~
data "aws_instances" "new" {
  instance_tags = {
    Name = var.project
   }
}
~~~

## Create main.tf file

~~~
# -----------------------------------------------------------
# ssh key pair
# -----------------------------------------------------------
resource "aws_key_pair" "sshkey" {

  key_name   = "${var.project}-key"
  public_key = file("./terrakey.pub")
  tags = {
    Name = "${var.project}-key"
    project = var.project
  }
}
# -----------------------------------------------------------
# Security Group For Webserver Access
# -----------------------------------------------------------
resource "aws_security_group" "webserver" {
  name        = "webserver"
  description = "allows all port conntection"
  ingress {
    description      = ""
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
  }
  ingress {
    description      = ""
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
  }
  tags = {
    Name = "${var.project}"
    project = var.project
  }
}
# -----------------------------------------------------------
# Classic Loadbalncer
# -----------------------------------------------------------
resource "aws_elb" "clb" {
  name_prefix        = "${substr(var.project, 0, 5)}-"
  security_groups    = [aws_security_group.webserver.id]
  availability_zones = [ "ap-south-1a", "ap-south-1b" ]
  listener {
    instance_port     = 80
    instance_protocol = "http"
    lb_port           = 80
    lb_protocol       = "http"
  }
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 2
    target              = "HTTP:80/health.html"
    interval            = 15
  }
  cross_zone_load_balancing   = true
  idle_timeout                = 60
  connection_draining         = true
  connection_draining_timeout = 5
  tags   = {
    Name = var.project
    project = var.project
  }
}

# -----------------------------------------------------------
# Launch Configuration
# -----------------------------------------------------------
resource "aws_launch_configuration" "frontend" {
  name_prefix     = "${var.project}-"
  key_name = aws_key_pair.sshkey.id
  image_id        = "${var.instance_ami}"
  instance_type   = "t2.micro"
  user_data       = file("setup.sh")
  security_groups = [aws_security_group.webserver.id]
  lifecycle {
    create_before_destroy = true
  }
}

# -----------------------------------------------------------
# AutoScaling Group
# -----------------------------------------------------------
resource "aws_autoscaling_group" "frontend" {
  name                      = "${var.project}"
  launch_configuration      = aws_launch_configuration.frontend.name
  min_size                  = 2
  max_size                  = 2
  desired_capacity          = 2
  health_check_grace_period = 120
  availability_zones = [ "ap-south-1a", "ap-south-1b" ]
  load_balancers            = [ aws_elb.clb.id ]
  wait_for_elb_capacity     = 2
  health_check_type         = "ELB"
  tag {
    key                 = "Name"
    value               = var.project
    propagate_at_launch = true
  }
  tag {
    key                 = "project"
    value               = var.project
    propagate_at_launch = true
  }
  lifecycle {
    create_before_destroy = true
  }
}

output "instance" {
    value = "http://${aws_elb.clb.dns_name}"
    value = "data.aws_instances.new.public_ips"
}

[ec2-user@ip-172-31-42-244 myproject]$ cat provider.tf
provider "aws" {
  region = var.region
  access_key = var.access_key
  secret_key = var.secret_key
}
~~~

Now the terraform part is completed and now we can start ansible part, we can fetch/call public IP of terraform created instance using ""TAG"" and ""Terraform output""

## Create ansible file main.yml

***Method 1.  Calling with TAG***

~~~
---
- name: "Creating Aws Infra Using Ansible"
  hosts: localhost
  become: true
  vars:
    access_key: " your access-key "
    secret_key: " your secret-key "
    region: "ap-south-1"
  tasks:

    - name: Basic deploy of a service
      community.general.terraform:
        project_path: 'myproject/'
        state: present
        force_init: true

    - name: "Amazon - Fetching Ec2 Info"
      amazon.aws.ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
            "tag:aws:autoscaling:groupName" : "new"
      register: ec2

    - name: "Amazon - Creating Dynamic Inventory"
      add_host:
        hostname: '{{ item.public_ip_address }}'
        ansible_host: '{{ item.public_ip_address }}'
        ansible_port: 22
        groups:
          - backends
        ansible_ssh_private_key_file: "./terrakey"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items: "{{ ec2.instances }}"


- name: "Deployment From GitHub"
  hosts: backends
  become: true
  serial: 1
  vars:
    packages:
      - git
    repo: https://github.com/devasivaram/aws-elb-site.git
  tasks:

    - name: "Package Installation"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Clonning Github Repository {{ repo }}"
      git:
        repo: "{{ repo }}"
        dest: "/var/website/"
      register: gitstatus

    - name: "Backend off loading from elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0000

    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 30

    - name: "updating site contents"
      when: gitstatus.changed
      copy:
        src: "/var/website/"
        dest: "/var/www/html/"
        remote_src: true
        owner: apache
        group: apache

    - name: "loading backend to elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0644

    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 20
~~~

***Method 2. Calling with output state of terraform

~~~
---
- name: "Creating Aws Infra Using Ansible"
  hosts: localhost
  become: true
  vars:
    access_key: " your access-key "
    secret_key: " your secret-key "
    region: "ap-south-1"
  tasks:

    - name: Basic deploy of a service
      community.general.terraform:
        project_path: 'myproject/'
        state: present
        force_init: true
      register: terra
      
    - name: "Creating Dynamic Inventory"
      add_host:
        hostname: '{{ item }}'
        groups: "terraform"
        ansible_host: '{{ item }}'
        ansible_port: 22
        ansible_ssh_private_key_file: "./key/terrakey"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ terra.outputs.instances.value }}"
 
- name: "Deployment From GitHub"
  hosts: terraform
  become: true
  serial: 1
  vars:
    packages:
      - git
    repo: https://github.com/devasivaram/aws-elb-site.git
  tasks:

    - name: "Package Installation"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Clonning Github Repository {{ repo }}"
      git:
        repo: "{{ repo }}"
        dest: "/var/website/"
      register: gitstatus

    - name: "Backend off loading from elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0000

    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 30

    - name: "updating site contents"
      when: gitstatus.changed
      copy:
        src: "/var/website/"
        dest: "/var/www/html/"
        remote_src: true
        owner: apache
        group: apache

    - name: "loading backend to elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0644

    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 20
 ~~~
  
  ***Once the main file is completed, run the main file using:***
  ~~~
  
  ~~~
  

### Conclusion
This is a classic load balancer with provisioning using ansible and terraform with serial flag. Please contact me when you encounter any difficulty error while using this terrform code. Thank you and have a great day!


### ⚙️ Connect with Me
<p align="center">
<a href="https://www.instagram.com/dev_anand__/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dev-anand-477898201/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>
