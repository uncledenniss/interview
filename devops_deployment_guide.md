# DevOps Deployment and Scaling Guide

## 3. Dockerized Laravel Application CI/CD with Jenkins

### Docker Setup

**Dockerfile for Laravel:**
```dockerfile
FROM php:8.2-fpm-alpine

# Install system dependencies
RUN apk add --no-cache \
    git \
    curl \
    libpng-dev \
    libxml2-dev \
    zip \
    unzip \
    nodejs \
    npm

# Install PHP extensions
RUN docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath gd

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www

# Copy application files
COPY . /var/www

# Install dependencies
RUN composer install --optimize-autoloader --no-dev
RUN npm install && npm run build

# Set permissions
RUN chown -R www-data:www-data /var/www
RUN chmod -R 755 /var/www/storage

EXPOSE 9000
CMD ["php-fpm"]
```

**Docker Compose for Development:**
```yaml
version: '3.8'
services:
  app:
    build: .
    container_name: laravel-app
    volumes:
      - .:/var/www
      - ./docker/php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - laravel

  nginx:
    image: nginx:alpine
    container_name: laravel-nginx
    ports:
      - "8000:80"
    volumes:
      - .:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
    networks:
      - laravel

  mysql:
    image: mysql:8.0
    container_name: laravel-mysql
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: laravel
      MYSQL_PASSWORD: secret
    ports:
      - "3306:3306"
    networks:
      - laravel

networks:
  laravel:
    driver: bridge
```

### Jenkins Pipeline Configuration

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'your-registry.com'
        IMAGE_NAME = 'laravel-app'
        KUBECONFIG = credentials('kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Environment Setup') {
            steps {
                script {
                    env.BUILD_VERSION = "${BUILD_NUMBER}-${GIT_COMMIT[0..7]}"
                    env.IMAGE_TAG = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_VERSION}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    composer install --no-dev --optimize-autoloader
                    npm ci
                    npm run production
                '''
            }
        }
        
        stage('Run Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            cp .env.testing .env
                            php artisan key:generate
                            php artisan config:clear
                            ./vendor/bin/phpunit --testsuite=Unit --coverage-text
                        '''
                    }
                }
                
                stage('Feature Tests') {
                    steps {
                        sh '''
                            php artisan migrate:fresh --seed --env=testing
                            ./vendor/bin/phpunit --testsuite=Feature
                        '''
                    }
                }
                
                stage('Code Quality') {
                    steps {
                        sh '''
                            ./vendor/bin/phpcs --standard=PSR2 app/
                            ./vendor/bin/phpstan analyse app/ --level=5
                        '''
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_TAG}")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh '''
                    # Vulnerability scanning
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --severity HIGH,CRITICAL ${IMAGE_TAG}
                '''
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        docker.image("${IMAGE_TAG}").push()
                        docker.image("${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh '''
                    helm upgrade --install laravel-staging ./helm/laravel \
                        --namespace staging \
                        --set image.tag=${BUILD_VERSION} \
                        --set environment=staging
                '''
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh '''
                    helm upgrade --install laravel-prod ./helm/laravel \
                        --namespace production \
                        --set image.tag=${BUILD_VERSION} \
                        --set environment=production \
                        --set replicaCount=3
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "✅ Pipeline succeeded for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "❌ Pipeline failed for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### Deployment Strategies

**Blue-Green Deployment:**
```yaml
# Blue environment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel
      version: blue
  template:
    spec:
      containers:
      - name: laravel
        image: your-registry.com/laravel-app:v1.0.0
```

## 4. Scalable AWS Stack Design and Management

### Architecture Overview

**Multi-Tier Architecture:**
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   CloudFront    │────│   Application    │────│    Database     │
│   + Route53     │    │  Load Balancer   │    │      RDS        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                              │
                    ┌─────────┼─────────┐
                    │         │         │
               ┌────▼───┐ ┌───▼───┐ ┌───▼───┐
               │  ECS   │ │  ECS  │ │  ECS  │
               │ Task 1 │ │ Task 2│ │ Task 3│
               └────────┘ └───────┘ └───────┘
```

### Infrastructure as Code (Terraform)

**Main Configuration:**
```hcl
# terraform/main.tf
provider "aws" {
  region = var.aws_region
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = true
  
  tags = var.common_tags
}

# RDS Database
resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-db"
  
  engine         = "postgres"
  engine_version = "14.9"
  instance_class = "db.t3.medium"
  
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = false
  final_snapshot_identifier = "${var.project_name}-final-snapshot"
  
  tags = var.common_tags
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = var.common_tags
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets
  
  enable_deletion_protection = true
  
  tags = var.common_tags
}
```

### ECS Service Configuration

**ECS Task Definition:**
```json
{
  "family": "laravel-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "laravel-app",
      "image": "your-registry.com/laravel-app:latest",
      "portMappings": [
        {
          "containerPort": 9000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:ssm:region:account:parameter/laravel/db/password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laravel-app",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Lambda Functions for Automation

**Auto Scaling Lambda:**
```python
import json
import boto3
import os
from datetime import datetime

def lambda_handler(event, context):
    ecs = boto3.client('ecs')
    cloudwatch = boto3.client('cloudwatch')
    
    cluster_name = os.environ['CLUSTER_NAME']
    service_name = os.environ['SERVICE_NAME']
    
    # Get current CPU utilization
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/ECS',
        MetricName='CPUUtilization',
        Dimensions=[
            {
                'Name': 'ServiceName',
                'Value': service_name
            },
            {
                'Name': 'ClusterName',
                'Value': cluster_name
            }
        ],
        StartTime=datetime.utcnow() - timedelta(minutes=5),
        EndTime=datetime.utcnow(),
        Period=300,
        Statistics=['Average']
    )
    
    if response['Datapoints']:
        avg_cpu = response['Datapoints'][0]['Average']
        
        # Scale up if CPU > 70%
        if avg_cpu > 70:
            scale_service(ecs, cluster_name, service_name, 'up')
        # Scale down if CPU < 30%
        elif avg_cpu < 30:
            scale_service(ecs, cluster_name, service_name, 'down')
    
    return {
        'statusCode': 200,
        'body': json.dumps('Auto scaling check completed')
    }

def scale_service(ecs, cluster, service, direction):
    current_service = ecs.describe_services(
        cluster=cluster,
        services=[service]
    )
    
    current_count = current_service['services'][0]['desiredCount']
    
    if direction == 'up':
        new_count = min(current_count + 1, 10)  # Max 10 tasks
    else:
        new_count = max(current_count - 1, 2)   # Min 2 tasks
    
    ecs.update_service(
        cluster=cluster,
        service=service,
        desiredCount=new_count
    )
```

### SNS/SQS Integration

**SQS Queue Configuration:**
```hcl
resource "aws_sqs_queue" "email_queue" {
  name                      = "${var.project_name}-email-queue"
  delay_seconds             = 90
  max_message_size          = 2048
  message_retention_seconds = 86400
  receive_wait_time_seconds = 10
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.email_dlq.arn
    maxReceiveCount     = 4
  })
  
  tags = var.common_tags
}

resource "aws_sns_topic" "alerts" {
  name = "${var.project_name}-alerts"
  
  tags = var.common_tags
}

resource "aws_sns_topic_subscription" "email_alerts" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

### Monitoring and Alerting

**CloudWatch Alarms:**
```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.project_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    ServiceName = aws_ecs_service.main.name
    ClusterName = aws_ecs_cluster.main.name
  }
}
```

## 5. DigitalOcean Scaling Strategy

### Droplet Management

**Ansible Playbook for Droplet Setup:**
```yaml
---
- name: Setup Laravel Application Servers
  hosts: webservers
  become: yes
  vars:
    app_name: laravel-app
    app_user: laravel
    
  tasks:
    - name: Update system packages
      apt:
        update_cache: yes
        upgrade: dist
        
    - name: Install required packages
      apt:
        name:
          - nginx
          - php8.1-fpm
          - php8.1-mysql
          - php8.1-xml
          - php8.1-curl
          - php8.1-mbstring
          - mysql-client
          - supervisor
          - certbot
        state: present
        
    - name: Create application user
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        home: "/home/{{ app_user }}"
        
    - name: Setup nginx configuration
      template:
        src: nginx-laravel.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
      notify: restart nginx
      
    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/{{ app_name }}
        dest: /etc/nginx/sites-enabled/{{ app_name }}
        state: link
      notify: restart nginx
```

### Load Balancer Configuration

**Terraform for DigitalOcean Load Balancer:**
```hcl
resource "digitalocean_loadbalancer" "public" {
  name   = "${var.project_name}-lb"
  region = var.region
  
  forwarding_rule {
    entry_protocol  = "https"
    entry_port      = 443
    target_protocol = "http"
    target_port     = 80
    certificate_name = digitalocean_certificate.cert.name
  }
  
  forwarding_rule {
    entry_protocol  = "http"
    entry_port      = 80
    target_protocol = "http"
    target_port     = 80
  }
  
  healthcheck {
    protocol               = "http"
    port                   = 80
    path                   = "/health"
    check_interval_seconds = 10
    response_timeout_seconds = 5
    unhealthy_threshold    = 3
    healthy_threshold      = 2
  }
  
  droplet_ids = digitalocean_droplet.web.*.id
}
```

### Auto-Scaling with Terraform

**Auto-Scaling Droplets:**
```hcl
resource "digitalocean_droplet" "web" {
  count  = var.droplet_count
  image  = var.droplet_image
  name   = "${var.project_name}-web-${count.index + 1}"
  region = var.region
  size   = var.droplet_size
  
  ssh_keys = [digitalocean_ssh_key.terraform.fingerprint]
  
  user_data = templatefile("${path.module}/cloud-init.yaml", {
    app_name = var.project_name
  })
  
  tags = ["web", "production", var.project_name]
}

# Database droplet
resource "digitalocean_droplet" "database" {
  image  = "ubuntu-20-04-x64"
  name   = "${var.project_name}-db"
  region = var.region
  size   = "s-2vcpu-4gb"
  
  ssh_keys = [digitalocean_ssh_key.terraform.fingerprint]
  
  tags = ["database", "production", var.project_name]
}
```

### Spaces (Object Storage) Integration

**Backup Script:**
```bash
#!/bin/bash
# backup-to-spaces.sh

SPACE_NAME="myapp-backups"
SPACE_REGION="nyc3"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)

# Database backup
mysqldump -u ${DB_USER} -p${DB_PASSWORD} ${DB_NAME} | gzip > /tmp/db_backup_${BACKUP_DATE}.sql.gz

# Application files backup
tar -czf /tmp/app_backup_${BACKUP_DATE}.tar.gz /var/www/laravel --exclude="vendor" --exclude="node_modules"

# Upload to Spaces using s3cmd
s3cmd put /tmp/db_backup_${BACKUP_DATE}.sql.gz s3://${SPACE_NAME}/database/
s3cmd put /tmp/app_backup_${BACKUP_DATE}.tar.gz s3://${SPACE_NAME}/application/

# Cleanup old backups (keep last 30 days)
find /tmp -name "*backup*" -type f -mtime +30 -delete

# Remove backups older than 90 days from Spaces
s3cmd ls s3://${SPACE_NAME}/database/ | while read line; do
  createDate=$(echo $line | awk '{print $1" "$2}')
  createDate=$(date -d "$createDate" +%s)
  olderThan=$(date -d "90 days ago" +%s)
  if [[ $createDate -lt $olderThan ]]; then
    fileName=$(echo $line | awk '{print $4}')
    s3cmd del "$fileName"
  fi
done
```

### Monitoring Setup

**Prometheus Configuration:**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['10.0.0.1:9100', '10.0.0.2:9100', '10.0.0.3:9100']
        
  - job_name: 'nginx-exporter'
    static_configs:
      - targets: ['10.0.0.1:9113', '10.0.0.2:9113', '10.0.0.3:9113']
        
  - job_name: 'mysql-exporter'
    static_configs:
      - targets: ['10.0.0.4:9104']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

**Alert Rules:**
```yaml
# alert_rules.yml
groups:
- name: infrastructure
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      
  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      
  - alert: DiskSpaceLow
    expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 85
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Disk space low on {{ $labels.instance }}"
```

### Deployment Automation

**Deployment Script:**
```bash
#!/bin/bash
# deploy.sh

set -e

ENVIRONMENT=${1:-staging}
VERSION=${2:-latest}

echo "Deploying ${VERSION} to ${ENVIRONMENT}..."

# Pull latest code
ansible-playbook -i inventory/${ENVIRONMENT} playbooks/deploy.yml \
  --extra-vars "app_version=${VERSION}" \
  --extra-vars "environment=${ENVIRONMENT}"

# Run health checks
ansible-playbook -i inventory/${ENVIRONMENT} playbooks/health-check.yml

# Update load balancer if needed
if [[ "$ENVIRONMENT" == "production" ]]; then
  terraform apply -target=digitalocean_loadbalancer.public -auto-approve
fi

echo "Deployment completed successfully!"
```
