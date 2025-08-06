# Provisioning secure EC2, CloudWatch Monitoring and S3 Backup with Terraform

## Introduction
Provisioning a secure Ubuntu EC2 instance with:
a. CloudWatch monitoring
b. S3 backup bucket
c. User-data Bash script to install Nginx

All using free-tier eligible services in us-east-1.

Folder Structure

ec2-s3-cloudwatch
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
├── user-data.sh
├── README.md


### 1. Creating Terraform configuration files

**Note** create directory and change to that directory.
``` mkdir name of directory
    cd name of directory
```

a. Create **provider.tf** file and the following below: 

```
provider "aws" {
  region = var.aws_region
}
```

<img width="635" height="203" alt="Image" src="https://github.com/user-attachments/assets/a3fd4adc-8546-4ac2-af8f-085b11eab45d" />


b. Create **main.tf** file and the following below: 

```
resource "random_id" "bucket_id" {
  byte_length = 4
}

resource "aws_s3_bucket" "backup_bucket" {
  bucket = "${var.bucket_prefix}${random_id.bucket_id.hex}"
  tags = {
    Name = "EC2BackupBucket"
  }
}

resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.backup_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.backup_bucket.bucket
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_cloudwatch_log_group" "ec2_logs" {
  name              = "/ec2/nginx"
  retention_in_days = 7
}

resource "aws_vpc" "default" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "default"
  }
}

resource "aws_security_group" "ec2_sg" {
  name        = "ec2_sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.default.id

  dynamic "ingress" {
    for_each = [
      {
        description = "HTTP"
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      },
      {
        description = "SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    ]

    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]
  user_data              = file("user-data.sh")
  subnet_id = aws_subnet.default.id

  tags = {
    Name = "SecureUbuntuEC2"
  }

  monitoring = true

  lifecycle {
    ignore_changes = [ami] # Optional: Keep stable AMI
  }
}

resource "aws_subnet" "default" {
  cidr_block = "10.0.1.0/24"
  vpc_id     = aws_vpc.default.id
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "default"
  }
}

resource "aws_internet_gateway" "default" {
  vpc_id = aws_vpc.default.id
  tags = {
    Name = "default"
  }
}

resource "aws_route_table" "default" {
  vpc_id = aws_vpc.default.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.default.id
  }
  tags = {
    Name = "default"
  }
}

resource "aws_route_table_association" "default" {
  subnet_id      = aws_subnet.default.id
  route_table_id = aws_route_table.default.id
}

```
<img width="647" height="636" alt="Image" src="https://github.com/user-attachments/assets/09a2bdf9-4287-456c-a48c-cd76e6dc6d8a" />

c. Create and configure variables.tf file

```
variable "aws_region" {
  default = "us-east-1"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "bucket_prefix" {
  default = "ec2-nginx-backup-"
}

variable "key_name" {
  default = "my-key"
}
```

<img width="643" height="418" alt="Image" src="https://github.com/user-attachments/assets/2bb9b9d6-be6a-4a0c-ad1b-ff2b6f69af80" />



d. Create and Configure Outputs.tf file

```
output "public_ip" {
  value = aws_instance.web_server.public_ip
}

output "s3_bucket_name" {
  value = aws_s3_bucket.backup_bucket.bucket
}
```
<img width="641" height="291" alt="Image" src="https://github.com/user-attachments/assets/ad65a903-238c-4662-834d-350b285850b4" />

e. Create and Configure user-data.sh file

```
#!/bin/bash
sudo apt update -y
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
echo "Welcome to My Terraform-Deployed EC2 Instance!" > /var/www/html/index.html
```

<img width="642" height="200" alt="Image" src="https://github.com/user-attachments/assets/c71a28d2-9d5a-4e99-b83e-5495ee1f0b34" />
### 2. Initialize Terraform
Run

```
terraform init
```

<img width="706" height="630" alt="Image" src="https://github.com/user-attachments/assets/f3e1af5f-45e7-4e99-bea0-2e7e2ecc7898" />

---

### 3. Validate & Plan

Run

```
terraform validate
```

<img width="485" height="74" alt="Image" src="https://github.com/user-attachments/assets/1c94fa9a-3993-4ba9-8775-3df03aeb409c" />

Run

```
terraform plan
```

<img width="707" height="709" alt="Image" src="https://github.com/user-attachments/assets/95e23dfe-67c5-45ea-b96e-08cb2cc35b4d" />


---

### 4. Deploy Resources


```
terraform apply
```
Type and enter **yes** when prompted.

<img width="707" height="709" alt="Image" src="https://github.com/user-attachments/assets/95e23dfe-67c5-45ea-b96e-08cb2cc35b4d" />>


---

### 5. Visit the Public IP
Terraform will print something like:

Outputs:
public_ip = "IP address"

**Test It**
After running terraform apply, copy the output public IP and open it in your browser. You should see the Nginx welcome page.


<img width="599" height="152" alt="Image" src="https://github.com/user-attachments/assets/f8b46427-c581-4b9d-9a0e-aa150a7b61e6" />


---

###. 6. Confirm resources provisioned on terraform

Run

```
terraform state list
```
and
```
terraform show
```

<img width="483" height="278" alt="Image" src="https://github.com/user-attachments/assets/5e62ee81-b8a3-431a-9c40-ef1c831fed0d" />


---

### 7. Cleanup (Avoid Charges)
a. To destroy all provisioned infrastructure:


Run
```
terraform destroy
```

<img width="708" height="695" alt="Image" src="https://github.com/user-attachments/assets/f8318788-ef70-4e60-9136-1b76f444ee5f" />

b. Run the following to confirm
bash
```
terraform state list
```
and
```
terraform show
```




---

###. 6. Confirm resources provisioned and destroyed from the Console

<img width="1138" height="238" alt="Image" src="https://github.com/user-attachments/assets/58f69879-a8c8-4ba1-99ef-0489c93b026f" />

![Image](https://github.com/user-attachments/assets/dd7f74d1-6d06-42bd-adc7-fff8d41f60fe)

![Image](https://github.com/user-attachments/assets/c514cdcd-8cfa-43ae-909e-d0fd8cff8772)

![Image](https://github.com/user-attachments/assets/bcf4af02-fc3e-4c5e-b459-f04583d6154a)

![Image](https://github.com/user-attachments/assets/559f3739-0f25-4094-ac57-30fe2c4a96ae)
