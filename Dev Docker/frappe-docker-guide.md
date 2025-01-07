# Frappe Docker Development Setup Guide

This guide will help you set up a Frappe development environment using Docker on Windows.

## Prerequisites

1. Install Docker Desktop for Windows
   - Download from https://www.docker.com/products/docker-desktop/
   - Run the installer
   - Restart your computer when prompted

2. Enable WSL 2 (Windows Subsystem for Linux)
   ```powershell
   wsl --install
   ```
   Restart your computer after installation.

## Setup Steps

1. Create the project directory structure:
   ```powershell
   cd C:\Users\YourUsername
   mkdir frappe-docker
   cd frappe-docker
   mkdir scripts
   mkdir frappe-bench
   ```

2. Create `docker-compose.yml` in the frappe-docker directory:
   ```yaml
   # docker-compose.yml
   services:
     mariadb:
       image: mariadb:10.6
       command:
         - --character-set-server=utf8mb4
         - --collation-server=utf8mb4_unicode_ci
         - --skip-character-set-client-handshake
         - --skip-innodb-read-only-compressed
       environment:
         MYSQL_ROOT_PASSWORD: 123
         MYSQL_ROOT_HOST: '%'
       volumes:
         - mariadb-data:/var/lib/mysql
       ports:
         - "3307:3306"

     redis-cache:
       image: redis:alpine
       ports:
         - "13000:6379"

     redis-queue:
       image: redis:alpine
       ports:
         - "11000:6379"

     redis-socketio:
       image: redis:alpine
       ports:
         - "12000:6379"

     frappe:
       image: frappe/bench:latest
       command: sleep infinity
       user: "1000:1000"
       environment:
         - SHELL=/bin/bash
       volumes:
         - ./scripts:/workspace/scripts
         - ./frappe-bench:/workspace/frappe-bench
       working_dir: /workspace
       ports:
         - "8000-8005:8000-8005"
         - "9000-9005:9000-9005"
       depends_on:
         - mariadb
         - redis-cache
         - redis-queue
         - redis-socketio

   volumes:
     mariadb-data:
   ```

3. Create `init.sh` in the scripts directory:
   ```bash
   #!/bin/bash

   set -e

   # Setup NodeJS
   source /home/frappe/.nvm/nvm.sh
   nvm alias default 18
   nvm use 18
   echo "nvm use 18" >> ~/.bashrc

   # Wait for MariaDB to be ready
   echo "Waiting for MariaDB to be ready..."
   until mysql -h mariadb -u root -p123 -e "SELECT 1" >/dev/null 2>&1
   do
       echo "MariaDB is unavailable - sleeping"
       sleep 1
   done

   echo "MariaDB is up - proceeding with setup"

   # Initialize Frappe Bench
   cd /workspace
   sudo chown -R frappe:frappe frappe-bench
   bench init \
   --ignore-exist \
   --skip-redis-config-generation \
   frappe-bench

   cd frappe-bench

   # Configure Redis and MariaDB hosts
   bench set-mariadb-host mariadb
   bench set-redis-cache-host redis-cache:6379
   bench set-redis-queue-host redis-queue:6379
   bench set-redis-socketio-host redis-socketio:6379

   # Remove redis from Procfile
   sed -i '/redis/d' ./Procfile

   # Create new site
   echo "Creating new site..."
   bench new-site dev.localhost \
   --mariadb-root-password 123 \
   --admin-password admin \
   --no-mariadb-socket

   bench --site dev.localhost set-config developer_mode 1
   bench --site dev.localhost clear-cache
   bench use dev.localhost

   echo "Setup complete! You can now start developing with Frappe."
   ```

4. Fix line endings in init.sh using PowerShell:
   ```powershell
   $content = Get-Content -Path "scripts\init.sh" -Raw
   $content = $content -replace "`r`n", "`n"
   Set-Content -Path "scripts\init.sh" -Value $content -NoNewline
   ```

5. Start the containers:
   ```powershell
   docker-compose up -d
   ```

6. Run the initialization script:
   ```powershell
   docker-compose exec frappe bash /workspace/scripts/init.sh
   ```

## Access the Application

Once setup is complete:
- Frappe web interface: http://localhost:8000
- Frappe desk (backend): http://localhost:8000/app
- Login credentials:
  - Username: Administrator
  - Password: admin

## Useful Commands

```powershell
# Start containers
docker-compose up -d

# Stop containers
docker-compose down

# View logs
docker-compose logs -f

# Access frappe container
docker-compose exec frappe bash

# Start Frappe development server (inside container)
bench start

# Create a new app (inside container)
bench new-app my_custom_app

# Complete reset
docker-compose down -v
rm -r frappe-bench
mkdir frappe-bench
```

## Troubleshooting

1. If you see port conflicts (e.g., 3306 already in use):
   - Check the ports in docker-compose.yml
   - Modify the port mappings if needed
   - Default ports used:
     - MariaDB: 3307 (changed from 3306)
     - Redis Cache: 13000
     - Redis Queue: 11000
     - Redis SocketIO: 12000
     - Frappe: 8000-8005, 9000-9005

2. Permission issues:
   - Make sure the frappe-bench directory exists before starting containers
   - The user "1000:1000" should have write permissions

3. Line ending issues:
   - Always use LF (Linux) line endings for init.sh
   - Use the provided PowerShell command to fix line endings

4. MariaDB connection issues:
   - Wait for the "MariaDB is up" message in the initialization script
   - Check MariaDB logs: `docker-compose logs mariadb`

## Development Workflow

1. Access the development environment:
   ```bash
   docker-compose exec frappe bash
   cd frappe-bench
   ```

2. Start the development server:
   ```bash
   bench start
   ```

3. Create a new app:
   ```bash
   bench new-app myapp
   ```

4. Install app on your site:
   ```bash
   bench --site dev.localhost install-app myapp
   ```

Remember to commit your changes to version control regularly!