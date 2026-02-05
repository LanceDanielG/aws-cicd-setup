# AWS CI/CD Deployment Guide (UI & API) - Generic Edition

## :warning: Clarification: Infrastructure & CI/CD Guide
> Note: This guide provides a general reference for deploying full-stack applications using **Bitbucket Pipelines** and **AWS SSM**.

---

## 🏗️ Architecture Overview

> [!NOTE]
> This setup is based on **Angular** (UI) and **Laravel** (API), but can be adapted for other frameworks.

*   **Region:** **`ap-southeast-1` (Singapore)** for all resources (unless otherwise noted).
*   **UI (Frontend):** Angular application hosted on **Amazon S3** and distributed via **Amazon CloudFront**.
*   **API (Backend):** PHP/Laravel application on **Amazon EC2** (Ubuntu/Nginx).
*   **Deployment Method:** 
    *   **UI:** S3 Sync + CloudFront Invalidation.
    *   **API:** **AWS Systems Manager (SSM)** `send-command` to run remote shell scripts.
*   **CI/CD:** **Bitbucket Pipelines** using **Deployment Environments**.

---

## 🛠️ 1. UI Infrastructure Setup (AWS Console)

Follow this order: **Route 53 → ACM → S3 → CloudFront → Route 53 (A Record)**

---

### :one: Step 1: Create Route 53 Hosted Zone (UI)
1.  Go to **Route 53 > Hosted zones > Create hosted zone**.
2.  **Domain name**: Enter your UI subdomain, e.g., `uat.yourdomain.com`.
3.  **Type**: Select **Public hosted zone**.
4.  Click **Create hosted zone**.
5.  **Copy the NS Records**: Add these 4 NS records to your **parent domain's registrar** (e.g., GoDaddy, Namecheap).

---

### :two: Step 2: Request ACM SSL Certificate (UI)
> [!IMPORTANT]
> For CloudFront, you **must** request the certificate in **`us-east-1` (N. Virginia)** region.

1.  Go to **AWS Certificate Manager (ACM)** in `us-east-1`.
2.  Click **Request a certificate > Request a public certificate**.
3.  **Domain name**: Enter `uat.yourdomain.com`.
4.  **Validation method**: Select **DNS validation**.
5.  Click **Request**.
6.  Click on the new certificate > Click **"Create records in Route 53"** to auto-add the CNAME validation record.
7.  Wait a few minutes until status changes to **Issued**.

---

### :three: Step 3: Create S3 Bucket (UI)
1.  Go to **S3 > Create bucket**.
2.  **Bucket name**: e.g., `my-app-ui-uat`.
3.  **Block Public Access**: Keep **"Block all public access"** enabled.
4.  Click **Create bucket**.
5.  **Bucket Policy**: Leave empty for now (CloudFront will provide one).

---

### :four: Step 4: Create CloudFront Distribution (UI)
1.  Go to **CloudFront > Create distribution**.
2.  **Origin**:
    *   **Origin domain**: Select your S3 bucket.
    *   **Origin access**: Select **Origin access control settings (OAC)** (Recommended) > Create new OAC > Click **Create**.
3.  **Default cache behavior**:
    *   **Viewer protocol policy**: Select **Redirect HTTP to HTTPS**.
    *   **Allowed HTTP methods**: `GET, HEAD` (standard for SPA).
    *   **Cache policy**: Use `CachingOptimized`.
4.  **Web Application Firewall (WAF)**:
    *   Select **Do not enable security protections** (to avoid additional WAF costs unless required).
5.  **Settings**:
    *   **Price class**: Select **Use only North America, Europe, Asia, Middle East, and Africa** (includes Singapore).
    *   **Alternate domain names (CNAMEs)**: Add `uat.yourdomain.com`.
    *   **Custom SSL certificate**: Select the ACM certificate you created in Step 2.
    *   **Default root object**: Set to `index.html`.
6.  Click **Create distribution**.
7.  **Update S3 Bucket Policy**: After creation, a banner will appear. Click **"Copy policy"** and go to S3 > Bucket > Permissions > Bucket policy > **Edit** > Paste and **Save**.
8.  **Wait for Deployment**: The status will show "Deploying". Wait until it shows the date.
9.  **Custom Error Responses (for SPAs)**: Go to **Error pages** tab:
    *   Create error response: `403` → `/index.html` → `200`
    *   Create error response: `404` → `/index.html` → `200`

---

### :five: Step 5: Add Route 53 A Record (UI → CloudFront)
1.  Go back to **Route 53 > Hosted zones > `uat.yourdomain.com`**.
2.  Click **Create record**.
3.  **Record name**: Leave blank.
4.  **Record type**: **A**.
5.  Toggle **Alias** to **ON**.
6.  **Route traffic to**: Select **Alias to CloudFront distribution**.
7.  **Choose distribution**: Select your UI distribution.
8.  Click **Create records**.

---

## 🛠️ 2. API Infrastructure Setup (AWS Console)

---

### :one: Step 1: Create Route 53 Hosted Zone (API)
1.  Go to **Route 53 > Hosted zones > Create hosted zone**.
2.  **Domain name**: Enter your API subdomain, e.g., `uat-api.yourdomain.com`.
3.  **Type**: Select **Public hosted zone**.
4.  Click **Create hosted zone**.
5.  **Copy the NS Records**: Add these 4 NS records to your **parent domain's registrar**.

---

### :two: Step 2: Launch EC2 Instance
1.  Go to **EC2 > Launch Instance**.
2.  **AMI**: Ubuntu 22.04 LTS.
3.  **Instance Type**: `t3.medium` or higher.
4.  **Key pair**: Create or select an existing one.
5.  **Security Group**:
    - Allow **Port 80/443** from `0.0.0.0/0`.
    - Allow **Port 22** from your IP only.
6.  Click **Launch instance**.

---

### :three: Step 3: Allocate Elastic IP
1.  Go to **EC2 > Elastic IPs > Allocate Elastic IP address**.
2.  Click **Allocate**.
3.  Select the new IP > **Actions > Associate Elastic IP address**.
4.  Choose your API EC2 instance.

---

### :four: Step 4: Attach IAM Role (for SSM)
1.  Go to **IAM > Roles > Create role**.
2.  **Trusted entity**: AWS Service > EC2.
3.  **Permissions**: Attach `AmazonSSMManagedInstanceCore`.
4.  **Name**: e.g., `EC2-SSM-Role`.
5.  Create role.
6.  Go to **EC2 > Instances > Select your instance > Actions > Security > Modify IAM role**.
7.  Select the role you just created.

---

### :five: Step 5: Add Route 53 A Record (API → EC2)
1.  Go back to **Route 53 > Hosted zones > `uat-api.yourdomain.com`**.
2.  Click **Create record**.
3.  **Record name**: Leave blank.
4.  **Record type**: **A**.
5.  **Value**: Enter the **Elastic IP** of your EC2 instance.
6.  **TTL**: `300`.
7.  Click **Create records**.

---

### :six: Step 6: Create RDS Database
1.  Go to **RDS > Create database**.
2.  **Engine**: Choose **MySQL** or **PostgreSQL**.
3.  **Template**: Select **Free tier** (for testing) or **Production**.
4.  **DB Instance Identifier**: e.g., `my-app-uat-db`.
5.  **Master username**: e.g., `admin`.
6.  **Master password**: Set a strong password.
7.  **DB Instance Class**: `db.t3.micro` (Free Tier) or larger.
8.  **Storage**: 20 GB (default is fine for UAT).
9.  **Connectivity**:
    - **VPC**: Same VPC as your EC2.
    - **Public access**: **No** (recommended for security).
    - **VPC Security Group**: Create new or use existing.
10. Click **Create database**.

#### Configure RDS Security Group
1.  Go to **EC2 > Security Groups**.
2.  Find the security group attached to your RDS.
3.  **Edit inbound rules**:
    - **Type**: MySQL/Aurora (3306) or PostgreSQL (5432).
    - **Source**: Select the **Security Group** of your EC2 instance.
4.  Save rules.

#### Get RDS Endpoint
1.  Go to **RDS > Databases > Select your DB**.
2.  Copy the **Endpoint** (e.g., `my-app-uat-db.abc123.ap-southeast-1.rds.amazonaws.com`).

---

### :seven: Step 7: Configure Laravel .env File
SSH into your EC2 and configure the Laravel environment:

```bash
cd /var/www/html/your-api
sudo -u www-data cp .env.example .env
sudo -u www-data nano .env
```

Update the following values:

```env
APP_NAME="Your App Name"
APP_ENV=uat
APP_KEY=
APP_DEBUG=false
APP_URL=https://uat-api.yourdomain.com

DB_CONNECTION=mysql
DB_HOST=my-app-uat-db.abc123.ap-southeast-1.rds.amazonaws.com
DB_PORT=3306
DB_DATABASE=your_database_name
DB_USERNAME=admin
DB_PASSWORD=your_rds_password

CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync
```

Generate the application key:
```bash
sudo -u www-data php artisan key:generate
```

Run migrations:
```bash
sudo -u www-data php artisan migrate --force
```

---

## 🖥️ 3. Server Environment Setup (SSH into EC2)

### :one: Install Nginx & PHP 7.4
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install nginx php7.4-fpm php7.4-common php7.4-mysql php7.4-xml php7.4-xmlrpc php7.4-curl php7.4-gd php7.4-imagick php7.4-cli php7.4-dev php7.4-imap php7.4-mbstring php7.4-opcache php7.4-soap php7.4-zip php7.4-intl -y
```

### :two: Install Composer
```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### :three: Nginx Configuration Template
Create configuration at `/etc/nginx/sites-available/app.conf`:

```nginx
server {
    server_name uat-api.yourdomain.com;
    root /var/www/html/your-api/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";
    client_max_body_size 200M;
    
    proxy_connect_timeout 600;
    proxy_send_timeout 600;
    proxy_read_timeout 600;
    keepalive_timeout 600;
    
    index index.html index.htm index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 3000;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    listen 443 ssl; # managed by Certbot
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### :four: Install Certbot & Get SSL
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d uat-api.yourdomain.com
```

### :five: Initial App Setup
```bash
sudo mkdir -p /var/www/html/your-api
sudo chown -R www-data:www-data /var/www/html/your-api
sudo -u www-data git clone https://bitbucket.org/user/repo.git /var/www/html/your-api
```

---

## 🚀 5. CI/CD Pipelines (Bitbucket)

### UI Pipeline (`bitbucket-pipelines.yml`)
```yaml
pipelines:
  branches:
    uat/*:
      - step:
            name: Deploy UI to S3
            deployment: UAT
            script:
                - npm install --legacy-peer-deps
                - npx ng build --configuration=uat
                - pipe: atlassian/aws-s3-deploy:0.4.4
                  variables:
                      AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID_ENV
                      AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY_ENV
                      S3_BUCKET: $AWS_S3_BUCKET
                - pipe: atlassian/aws-cloudfront-invalidate:0.1.1
                  variables:
                      DISTRIBUTION_ID: $AWS_DISTRIBUTION_ID
```

### API Pipeline (`bitbucket-pipelines.yml`)
```yaml
image: amazon/aws-cli:2.17.57
pipelines:
  branches:
    uat/*:
      - step:
          name: Deploy API via SSM
          deployment: API-UAT
          script:
            - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            - aws configure set region $AWS_DEFAULT_REGION
            - |
              commands=(
                "cd /var/www/html/your-api && sudo -u www-data git pull origin $BITBUCKET_BRANCH"
                "cd /var/www/html/your-api && sudo -u www-data php artisan migrate --force"
                "cd /var/www/html/your-api && sudo -u www-data composer install --no-interaction --optimize-autoloader"
                "cd /var/www/html/your-api && sudo -u www-data php artisan cache:clear"
              )
            - | 
              for cmd in "${commands[@]}"; do
                command_id=$(aws ssm send-command --targets "Key=instanceIds,Values=$EC2_INSTANCE_ID" --document-name "AWS-RunShellScript" --parameters "{\"commands\":[\"$cmd\"]}" --query "Command.CommandId" --output text)
                
                while true; do
                  status=$(aws ssm get-command-invocation --command-id "$command_id" --instance-id "$EC2_INSTANCE_ID" --query 'Status' --output text)
                  if [ "$status" == "Success" ]; then break; fi
                  if [ "$status" == "Failed" ]; then exit 1; fi
                  sleep 5
                done
              done
```

---

## 🛠️ 6. Bitbucket Configuration (Step-by-Step)

### :one: Enable Pipelines
1.  Go to your **Repository > Repository settings**.
2.  Under **Pipelines**, click **Settings**.
3.  Toggle **Enable Pipelines** to ON.

---

### :two: Create Deployment Environments
1.  Go to **Repository settings > Deployments**.
2.  Click **Add environment**.
3.  Create the following environments:

| Environment Name | Used For |
| :--- | :--- |
| `UAT` | UI deployments to S3/CloudFront |
| `API-UAT` | API deployments via SSM |

4.  (Optional) Add **restrictions** like requiring manual approval or limiting to specific branches.

---

### :three: Add Repository Variables
1.  Go to **Repository settings > Pipelines > Repository variables**.
2.  Add the following variables:

| Variable Name | Value | Secured? |
| :--- | :--- | :---: |
| `AWS_ACCESS_KEY_ID` | IAM Key for SSM | ✅ Yes |
| `AWS_SECRET_ACCESS_KEY` | IAM Secret for SSM | ✅ Yes |
| `AWS_ACCESS_KEY_ID_ENV` | IAM Key for S3/CloudFront | ✅ Yes |
| `AWS_SECRET_ACCESS_KEY_ENV` | IAM Secret for S3/CloudFront | ✅ Yes |
| `AWS_DEFAULT_REGION` | e.g., `ap-southeast-1` | ❌ No |
| `EC2_INSTANCE_ID` | e.g., `i-0abc123def456` | ❌ No |
| `AWS_S3_BUCKET` | e.g., `my-app-ui-uat` | ❌ No |
| `AWS_DISTRIBUTION_ID` | CloudFront Distribution ID | ❌ No |

> [!TIP]
> You can also add variables to specific **Deployment environments** instead of the repository level for better isolation.

---

### :four: Create IAM User for Pipelines
1.  Go to **AWS IAM > Users > Create user**.
2.  **User name**: e.g., `bitbucket-pipeline-user`.
3.  **Permissions**: Attach the following policies:
    - `AmazonS3FullAccess` (or a custom S3 policy)
    - `CloudFrontFullAccess` (or custom invalidation policy)
    - Custom policy for SSM:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:SendCommand",
                    "ssm:GetCommandInvocation",
                    "ssm:ListCommandInvocations"
                ],
                "Resource": "*"
            }
        ]
    }
    ```
4.  **Create Access Key**: After creating the user, go to **Security credentials > Create access key**.
5.  Copy the **Access Key ID** and **Secret Access Key** into Bitbucket variables.

---

### :five: (Optional) SSH Keys for Git Operations
If your pipeline needs to clone private repos or push changes:
1.  Go to **Repository settings > Pipelines > SSH keys**.
2.  Click **Generate keys**.
3.  Copy the **Public key**.
4.  Add it to the target repository's **Access keys** or your Bitbucket/GitHub account.

---

### :six: (Optional) Known Hosts
If you're SSHing directly to EC2 (without SSM):
1.  Go to **Repository settings > Pipelines > SSH keys > Known hosts**.
2.  Click **Fetch** and enter your EC2 Elastic IP.
3.  Click **Add host**.
