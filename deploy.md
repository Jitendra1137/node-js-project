# Deploy Node.js Backend to AWS EC2 via GitHub Actions CI/CD

This guide explains how to deploy a Node.js backend to an EC2 instance using a fully automated GitHub Actions CI/CD workflow.

---

## Step 1: Update & Upgrade Packages

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 2: Install Node.js & npm

```bash
sudo apt-get install npm -y
sudo npm i -g n
sudo n lts
```

Logout & login again:

```bash
node -v
npm -v
```

---

## Step 3: Install Nginx

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

---

## Step 4: Create Deployment Directory

```bash
sudo mkdir -p /var/www/express-app
cd /var/www/express-app
```

(Optional)

```bash
sudo chown -R ubuntu:ubuntu /var/www/express-app
```

---

## Step 5: Configure Nginx Reverse Proxy

Create config:

```bash
sudo nano /etc/nginx/sites-available/express-app
```

Paste:

```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/express-app /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## Step 6: Install PM2

```bash
sudo npm install -g pm2
sudo pm2 startup
```

Start app:

```bash
sudo pm2 start ecosystem.config.cjs
sudo pm2 save
```

---

## Step 7: Add GitHub Secrets

Go to:

**GitHub → Repo → Settings → Secrets → Actions**

Add the following secrets:

* `EC2_HOST`
* `EC2_USERNAME`
* `EC2_SSH_KEY`

Copy your PEM key:

```bash
cat ec2-deploy-key.pem
```

---

## Step 8: GitHub Actions Workflow

Create file:
`.github/workflows/deploy.yml`

Paste:

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20.13.1"
          cache: "npm"

      - run: npm ci
      - run: npm run build

      - name: Create deployment package
        run: |
          mkdir -p deploy
          cp -r dist deploy/
          cp -r src deploy/
          cp package*.json deploy/
          cp ecosystem.config.cjs deploy/
          [ -f .env.example ] && cp .env.example deploy/
          tar -czf deploy.tar.gz -C deploy .

      - uses: actions/upload-artifact@v4
        with:
          name: deployment-package
          path: deploy.tar.gz
          retention-days: 1

  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/download-artifact@v4
        with:
          name: deployment-package

      - name: Deploy to EC2
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          scp -i private_key.pem -o StrictHostKeyChecking=no \
            deploy.tar.gz ${EC2_USERNAME}@${EC2_HOST}:/tmp/

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'

            cd /var/www/express-app
            sudo chown -R $USER:$USER /var/www/express-app

            if [ -d "dist" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mkdir -p backups
              tar -czf backups/backup_${timestamp}.tar.gz dist package.json ecosystem.config.cjs .env 2>/dev/null || true
              ls -t backups/backup_*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
            fi

            tar --overwrite -xzf /tmp/deploy.tar.gz -C /var/www/express-app
            rm /tmp/deploy.tar.gz

            sudo chown -R $USER:$USER /var/www/express-app

            npm ci

            if [ ! -f .env ]; then
              echo "NODE_ENV=production" > .env
              echo "PORT=8000" >> .env
            fi

            mkdir -p logs

            if pm2 describe express-app >/dev/null 2>&1; then
              sudo pm2 reload ecosystem.config.cjs --update-env
            else
              sudo pm2 start ecosystem.config.cjs
            fi

            sudo pm2 save
          EOF

          rm private_key.pem
```

---

## ✔ Your Deployment Pipeline Is Ready!

This file includes:

* EC2 setup
* Nginx configuration
* PM2 setup
* Full GitHub Actions CI/CD workflow
* Rollbacks & auto backup support

You can now push `deploy.md` to GitHub.
