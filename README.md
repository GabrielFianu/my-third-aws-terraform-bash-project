# Secure EC2 with Nginx, S3 Backup & CloudWatch (Terraform)

## Overview
This project provisions:
- Ubuntu EC2 instance (free tier)
- S3 bucket with encryption & versioning
- CloudWatch log group
- Nginx setup using `user-data.sh`

## Features
- Secure security group  
- User-data to auto-install Nginx  
- CloudWatch monitoring enabled  
- S3 backup bucket (encrypted)  
- Free-tier eligible (`us-east-1` region)

## How to Use
```bash
terraform init
terraform plan
terraform apply

## Notes
- All resources are Free Tier eligible.
- Tags help track and manage infrastructure.
