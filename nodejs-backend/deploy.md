## Deploy the Node.js backend to an EC2 instance using an automated GitHub Actions CI/CD workflow

After setting up the EC2 instance and configuring security groups, follow these steps to deploy the application:

#### Setp 1: First update & upgrade the packages on your EC2 instance:

```bash
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Install Node.js and npm on your EC2 instance:

```bash
sudo apt-get install npm -y
sudo npm i -g n
sudo n lts # sudo n 22.0.1
```

**After that exit your instance and relogin to check the new node js version**

#### Step 3: Now Install the Nginx server on your EC2 instance:

```bash
sudo apt install nginx -y

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx

```

#### Step 4: Setup Deployment Directory Structure

```bash
# Create app directory
sudo mkdir -p /var/www/express-app
# If you want to set ownership to ubuntu user
# sudo chown -R ubuntu:ubuntu /var/www/express-app
cd /var/www/express-app
```

#### Step 5: Configure Nginx as Reverse Proxy

```bash
# Create Nginx configuration file
sudo nano /etc/nginx/sites-available/express-app
```

##### Paste the following configuration:

```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;  # Replace with your IP or domain

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

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}

```

##### Enable the Configuration

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/express-app /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

#### Step 6: install PM2 to manage the Node.js application

```bash
sudo npm install -g pm2

## Generate the start script using PM2
sudo pm2 startup
```

After this run your application using PM2 in the deployment script and use this command to start the server with PM2:

```bash
sudo pm2 start ecosystem.config.js

## Save the PM2
sudo pm2 save
```

#### Step 7: Create GitHub Secrets for Deployment

In your GitHub repository, go to Settings → Secrets and variables → Actions → New repository secret.

Add These Secrets

**EC2_HOST - Your EC2 public IP address:**
**EC2_USERNAME - EC2 user (ubuntu for Ubuntu AMI):**
**EC2_SSH_KEY - Your private SSH key content:**

```bash
# On your local machine, copy the entire key
cat ec2-deploy-key.pem

# Copy ALL content including:
# -----BEGIN RSA PRIVATE KEY-----
# ... key content ...
# -----END RSA PRIVATE KEY-----
```

#### Step 8: Create GitHub Actions Workflow

In your repository, create .github/workflows/deploy.yml

```yml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch: # manual trigger

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20.13.1"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build TypeScript
        run: npm run build

      - name: Create deployment package (without node_modules)
        run: |
          mkdir -p deploy
          cp -r dist deploy/
          cp -r src deploy/
          cp package*.json deploy/
          cp ecosystem.config.cjs deploy/
          # Copy .env.example if exists
          [ -f .env.example ] && cp .env.example deploy/ || echo "No .env.example found"
          tar -czf deploy.tar.gz -C deploy .

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-package
          path: deploy.tar.gz
          retention-days: 1

  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-package

      - name: Deploy to EC2 via SSH
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          # Create SSH key file
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          # Copy deployment package to EC2
          scp -i private_key.pem -o StrictHostKeyChecking=no \
            deploy.tar.gz ${EC2_USERNAME}@${EC2_HOST}:/tmp/

          # SSH into EC2 and deploy
          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'
            
            # Navigate to app directory
            cd /var/www/express-app

            # Fix ownership first (important!)
            echo "Fixing directory ownership..."
            sudo chown -R $USER:$USER /var/www/express-app
            
            
            # Backup current version
            if [ -d "dist" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mkdir -p backups
              tar -czf backups/backup_${timestamp}.tar.gz dist package.json ecosystem.config.cjs .env 2>/dev/null || true
              # Keep only last 5 backups
              ls -t backups/backup_*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
            fi
            
            # Extract new version (this will overwrite existing files)
            echo "Extracting new deployment package..."
            tar --overwrite -xzf /tmp/deploy.tar.gz -C /var/www/express-app
            rm /tmp/deploy.tar.gz

            # Fix ownership after extraction
            sudo chown -R $USER:$USER /var/www/express-app
            
            
            # Install production dependencies on the server
            echo ""
            echo "Installing production dependencies..."
            npm ci
            
            # Create .env file if needed (only on first deploy)
            if [ ! -f .env ]; then
              echo "Creating default .env file..."
              echo "NODE_ENV=production" > .env
              echo "PORT=8000" >> .env
            fi
            
            # Create logs directory
            mkdir -p logs
            
            # Zero-downtime reload with PM2
            if pm2 describe express-app > /dev/null 2>&1; then
              echo "Reloading PM2 app (zero-downtime)..."
              sudo pm2 reload ecosystem.config.cjs --update-env
            else
              echo "Starting new PM2 process..."
              sudo pm2 start ecosystem.config.cjs
            fi
            
            # Save PM2 process list
            sudo pm2 save
           
          EOF

          # Cleanup
          rm private_key.pem

      - name: Verify deployment
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
        run: |
          echo "Waiting for application to start..."
          sleep 10

          # Check if app responds via nginx (try multiple times)
          for i in {1..2}; do
            echo "Attempt $i of 2..."
            response=$(curl -s -o /dev/null -w "%{http_code}" http://${EC2_HOST} 2>/dev/null || echo "000")
            
            if [ "$response" = "200" ] || [ "$response" = "301" ] || [ "$response" = "302" ]; then
              echo "✅ Deployment successful! App is responding with status: $response"
              exit 0
            fi
            
            echo "Got response: $response, waiting 5 seconds..."
            sleep 5
          done

          echo "❌ Deployment verification failed after 2 attempts"
          echo "Please check PM2 logs on the server"
          exit 1

      - name: Rollback on failure
        if: failure()
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "🔄 Attempting to rollback to previous version..."

          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'
            
            cd /var/www/express-app

            # Fix ownership first (important!)
            sudo chown -R $USER:$USER /var/www/express-app
            
            # Find latest backup
            latest_backup=$(ls -t backups/backup_*.tar.gz 2>/dev/null | head -1)
            
            if [ -n "$latest_backup" ]; then
              echo "Found backup: $latest_backup"
              tar -xzf "$latest_backup" -C /var/www/express-app
              npm ci
              pm2 reload ecosystem.config.cjs
              echo "✅ Rolled back to previous version"
            else
              echo "⚠️ No backup found, cannot rollback"
            fi
          EOF

          rm private_key.pem
```
