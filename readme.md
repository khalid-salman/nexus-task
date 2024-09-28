To solve this task, we'll break it down into three main steps: creating the Terraform infrastructure, setting up Ansible scripts for Nexus installation, and integrating everything with Jenkins using two agents (one for Terraform and one for Ansible).

### Step 1: Terraform Project for VPC, EC2, and Inventory File Creation
#### a. Initialize Terraform
1. **Create the Terraform configuration:**
   - Define a VPC, subnets, internet gateway, route tables, and security groups.
   - Create an EC2 instance of type "t2.medium" (or adjust based on your requirement).

#### Terraform Configuration
1. **Main `main.tf`:**
```hcl
provider "aws" {
  region = "us-east-1" # Set the desired AWS region
}

resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "my-vpc"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.my_vpc.id
}

resource "aws_route_table" "public_route" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
}

resource "aws_route_table_association" "public_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_route.id
}

resource "aws_security_group" "ec2_sg" {
  vpc_id = aws_vpc.my_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "my_ec2" {
  ami           = "ami-0c55b159cbfafe1f0" # Ubuntu AMI
  instance_type = "t2.medium"
  subnet_id     = aws_subnet.public_subnet.id
  security_groups = [aws_security_group.ec2_sg.name]

  tags = {
    Name = "Nexus-EC2"
  }

  user_data = <<-EOF
              #!/bin/bash
              echo "[nexus]" > /tmp/inventory
              echo "$(hostname -I | awk '{print $1}') ansible_ssh_user=ubuntu" >> /tmp/inventory
              EOF
}

output "ec2_public_ip" {
  value = aws_instance.my_ec2.public_ip
}
```
2. **Run the Terraform commands:**
   ```bash
   terraform init
   terraform apply
   ```

   This will create the VPC, subnets, and the EC2 instance. The inventory file will be created on the instance.

### Step 2: Ansible Scripts to Install Nexus
#### b. Create an Ansible playbook to install Nexus
1. **Create the `nexus.yml` playbook:**
```yaml
---
- hosts: nexus
  become: yes
  tasks:
    - name: Update and install Java
      apt:
        update_cache: yes
        name: openjdk-8-jdk
        state: present

    - name: Create Nexus user
      user:
        name: nexus
        system: yes
        shell: /bin/bash

    - name: Download Nexus
      get_url:
        url: "https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
        dest: "/opt/nexus.tar.gz"

    - name: Extract Nexus
      unarchive:
        src: "/opt/nexus.tar.gz"
        dest: "/opt"
        remote_src: yes

    - name: Set Nexus permissions
      file:
        path: /opt/nexus-*
        owner: nexus
        group: nexus
        state: directory
        recurse: yes

    - name: Configure Nexus as a service
      copy:
        content: |
          [Unit]
          Description=Nexus Service
          After=network.target

          [Service]
          Type=forking
          User=nexus
          Group=nexus
          ExecStart=/opt/nexus-*/bin/nexus start
          ExecStop=/opt/nexus-*/bin/nexus stop
          User=nexus
          LimitNOFILE=65536

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/nexus.service
        owner: root
        group: root
        mode: '0644'

    - name: Start and enable Nexus service
      systemd:
        name: nexus
        state: started
        enabled: yes
```
2. **Create an Ansible inventory:**
   The EC2 instance creates the inventory file in `/tmp/inventory`. You will copy this file to your Jenkins server for use with Ansible.

### Step 3: Jenkins Setup
#### c. Jenkinsfile and Docker Agents
1. **Create two agents using Dockerfiles:**

- **Terraform Agent `Dockerfile`:**
  ```Dockerfile
  FROM hashicorp/terraform:latest

  RUN apk add --no-cache bash curl
  ```
- **Ansible Agent `Dockerfile`:**
  ```Dockerfile
  FROM williamyeh/ansible:alpine3

  RUN apk add --no-cache openssh
  ```

2. **Jenkinsfile:**
```groovy
pipeline {
    agent none
    stages {
        stage('Terraform Init and Apply') {
            agent { docker { image 'terraform-agent-image' } }
            steps {
                sh 'terraform init'
                sh 'terraform apply -auto-approve'
            }
        }
        stage('Run Ansible Playbook') {
            agent { docker { image 'ansible-agent-image' } }
            steps {
                sh 'ansible-playbook -i /tmp/inventory nexus.yml'
            }
        }
    }
}
```

### Explanation:
- **Terraform Project:** You create a VPC, public EC2 instance, and an inventory file that Ansible will use to manage the EC2.
- **Ansible Playbook:** The playbook installs Nexus Repository Manager and configures it as a service on the EC2 instance.
- **Jenkins Pipeline:** It runs Terraform to set up the infrastructure and then uses Ansible to provision Nexus. The two different agents (Terraform and Ansible) ensure isolation of environments.

This setup automates the entire process from infrastructure creation to software installation and orchestration using Jenkins.