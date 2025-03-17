# **How to Deploy Nest js & Nextjs to Hostinger Vps & enable github actions CI/CD**

#  link the vps with domain
```
1- copy vps ip address 
2- go to domain dns system 
3- replace A record with 
@ vps ip address 3600(ttl)
```
# Link Any sub domain
```
#add a dns record for each subdomain
for example api.domain.com
name : api
type : A
ipe : vps ip
ttl :3600
```
# setup git & node envs to server  
```
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Node.js and npm (using Node.js 18.x as an example)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install build essentials
sudo apt install -y build-essential

# Install Git
sudo apt install -y git

# Install PM2 for process management
sudo npm install -g pm2

# Verify installations
node -v
npm -v
git --version
```

# Next.js Application Setup

```
# Create directory for your app
mkdir -p /var/www/nextjs-app
cd /var/www/nextjs-app

# Clone your Next.js repository (replace with your repo URL)
git clone https://github.com/yourusername/your-nextjs-app.git .
** if private ** 
-username
-password : get it from github deplovepers page (token)
# Install dependencies
npm install

nano .env
-> copy and paste then ctrl+x

# Build the app
npm run build

# Start with PM2
pm2 start npm --name "nextjs-app" -- start
```
# Nest.js Application Setup
```
# Create directory for your app
mkdir -p /var/www/nestjs-app
cd /var/www/nestjs-app

# Clone your NestJS repository (replace with your repo URL)
git clone https://github.com/yourusername/your-nestjs-app.git .

# Install dependencies
npm install
- if u use pnpm 
-> install -g pnpm 
then pnpm install

#add .env 

nano .env
-> copy and paste then ctrl+x

# Build the app
npm run build

# Start with PM2
pm2 start dist/main.js --name "nestjs-app"
```

# Setup Nginx as Reverse Proxy

```
# Install Nginx
sudo apt install -y nginx

# Configure Nginx for Next.js
sudo nano /etc/nginx/sites-available/nextjs

# Add configuration
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# Configure Nginx for NestJS
sudo nano /etc/nginx/sites-available/nestjs

# Add configuration
server {
    listen 80;
    server_name www.api.yourdomain.com api.yourdomain.com;

    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# Enable sites
sudo ln -s /etc/nginx/sites-available/nextjs /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/nestjs /etc/nginx/sites-enabled/

# Test and restart Nginx
sudo nginx -t
sudo systemctl restart nginx
```

# Add ssl certificates (https)
```
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificates
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
sudo certbot --nginx -d api.yourdomain.com

# Auto-renewal is set up by default

#open your nginx and test if ssl certificates is enabled
sudo nano /etc/nginx/sites-available/nestjs
sudo nano /etc/nginx/sites-available/nextjs
#then if added 
ctrl+x
```
# Add A github Workflow
Note : the secret keys u can add them to github repo 
from 
--> repo settings
---> secrets and keys
----> create secret repo
* host : vps ip 
* user : root or (username)
* ssh_key ---> cat ~/.ssh/id_rsa
```
# .github/workflows/nextjs-deploy.yml
name: Deploy Next.js Application

on:
  push:
    branches: [ main ]
    paths:
      - 'nextjs-app/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        working-directory: ./
        run: npm ci
        
      - name: Build application
        working-directory: ./
        run: npm run build
        
      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/nextjs-app
           git pull git@github.com:username/repo-name.git

            npm ci
            npm run build
            pm2 restart nextjs-app

# .github/workflows/nestjs-deploy.yml
name: Deploy NestJS Application

on:
  push:
    branches: [ main ]
    paths:
      - 'nestjs-app/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        working-directory: ./
        run: npm ci
        
      - name: Build application
        working-directory: ./
        run: npm run build
        
      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/nestjs-app
            git pull
            npm ci
            npm run build
            pm2 restart nestjs-app
```
