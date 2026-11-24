# 3-Tier Web Application Deployment on AWS (Linux)

A hands-on, production-grade 3-tier Linux infrastructure project deployed on AWS (`10.0.0.0/16` VPC). The design isolates the **Web Tier**, **Application Tier**, and **Database Tier** into dedicated subnets with strict security group chaining and Linux service orchestration.

---

## Architecture Overview

- **Web Tier (`three-tier-web`):** Public subnet (`10.0.1.0/24`). Runs Apache HTTP Server (`httpd`) acting as a reverse proxy forwarding HTTP requests from port `80` to the App tier on port `5000`.
- **Application Tier (`three-tier-app`):** Private subnet (`10.0.2.0/24`). Runs a Python Flask application (`/app/app.py`) listening on port `5000` with direct database connectivity via `pymysql`.
- **Database Tier (`three-tier-db`):** Private subnet (`10.0.3.0/24`). Runs MariaDB (`10.5`) on port `3306` with no direct internet access, serving the `student_db` database and `students` table.

---

## Network & Security Design

### Custom VPC (`three-tier-vpc` — `10.0.0.0/16`)
- **Public Subnet:** `web-public-subnet-1` (`10.0.1.0/24`) — Auto-assign public IP enabled.
- **Private Subnets:** `app-private-subnet-1` (`10.0.2.0/24`) and `db-private-subnet-1` (`10.0.3.0/24`).
- **Routing:**
  - `web-public-rt`: `0.0.0.0/0` -> Internet Gateway (`three-tier-igw`), associated with `web-public-subnet-1`.
  - `app-db-private-rt`: `10.0.0.0/16` local route only, associated with `app-private-subnet-1` and `db-private-subnet-1`.

### Least-Privilege Security Group Chaining
- **Web Security Group (`web-sg`):**
  - Inbound HTTP (`80`) from `0.0.0.0/0`.
  - Inbound SSH (`22`) from admin IP.
- **App Security Group (`app-sg`):**
  - Inbound Custom TCP (`5000`) exclusively from `web-sg`.
  - Inbound SSH (`22`) from `web-sg` or admin source.
- **DB Security Group (`db-sg`):**
  - Inbound MySQL/MariaDB (`3306`) exclusively from `app-sg`.
  - Inbound SSH (`22`) from admin/jump host.

---

## Key Administrative Techniques

1. **SSH Agent Forwarding (`App.pem`):** Access private App and DB servers securely without uploading private `.pem` keys to intermediate EC2 instances.
2. **Reverse Proxy Configuration:** Configured Apache's `mod_proxy` module (`flask_proxy.conf`) to proxy root paths directly to the internal Flask microservice.
3. **Database Schema Setup:** Provisioned a relational schema (`students`) supporting unique constraint checks, auto-increment keys, and non-null validation.

---

## Step-by-Step Deployment Commands

### 1. Local SSH Setup & Bastion Hopping (Git Bash)

```bash
# Prepare permissions for private key
chmod 400 App.pem

# Load SSH Agent and register private key locally
eval "$(ssh-agent -s)"
ssh-add App.pem

# Connect to Public Web Tier with Agent Forwarding (-A)
ssh -A ec2-user@<WEB-PUBLIC-IP>

# Hop from Web Server to Private App Server
ssh ec2-user@<APP-PRIVATE-IP>

# Hop from App Server to Private DB Server
ssh ec2-user@<DB-PRIVATE-IP>
```

---

### 2. Database Tier Configuration (`three-tier-db`)

```bash
# Install and start MariaDB 10.5
sudo dnf install -y mariadb105-server || sudo yum install -y mariadb-server
sudo systemctl enable --now mariadb
sudo ss -tulpn | grep 3306

# Create database, user, privileges, and students schema
sudo mysql <<'EOF'
CREATE DATABASE IF NOT EXISTS student_db;
CREATE USER IF NOT EXISTS 'flaskuser'@'%' IDENTIFIED BY 'FlaskPass123';
ALTER USER 'flaskuser'@'%' IDENTIFIED BY 'FlaskPass123';
GRANT ALL PRIVILEGES ON student_db.* TO 'flaskuser'@'%';
FLUSH PRIVILEGES;
USE student_db;
CREATE TABLE IF NOT EXISTS students (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  roll_no VARCHAR(50) NOT NULL UNIQUE,
  email VARCHAR(120) NOT NULL,
  dept VARCHAR(80) NOT NULL,
  age INT NOT NULL
);
SHOW TABLES;
EOF

# Verify table schema
sudo mysql -e "USE student_db; DESCRIBE students;"
```

---

### 3. Application Tier Configuration (`three-tier-app`)

```bash
# Create application directory structure
sudo mkdir -p /app
sudo chown -R ec2-user:ec2-user /app

# Install dependencies
sudo dnf install -y python3-pip || sudo yum install -y python3-pip
pip3 install flask pymysql

# Validate python script syntax
python3 -m py_compile /app/app.py

# Launch Flask application in background
nohup python3 /app/app.py > /tmp/flask.log 2>&1 &
sleep 1

# Verify port 5000 binding and runtime logs
cat /tmp/flask.log
sudo ss -tulpn | grep 5000
```

---

### 4. Web Tier Configuration (`three-tier-web`)

```bash
# Install and start Apache HTTP Server
sudo dnf install -y httpd || sudo yum install -y httpd
sudo systemctl enable --now httpd

# Create Reverse Proxy configuration file
sudo tee /etc/httpd/conf.d/flask_proxy.conf > /dev/null <<'EOF'
ProxyPreserveHost On
ProxyPass / http://<APP-PRIVATE-IP>:5000/
ProxyPassReverse / http://<APP-PRIVATE-IP>:5000/
EOF

# Test syntax and reload HTTP service
sudo httpd -t
sudo systemctl restart httpd

# Test proxy locally
curl -I http://localhost
```

---

## Verification & Testing

1. **Browser Access:** Open `http://<WEB-PUBLIC-IP>` in a browser to load the Student Registration Form.
2. **Data Submission:** Add a student record (Name, Roll Number, Email, Department, Age) via the web interface.
3. **Database Validation:** Query MariaDB directly on `three-tier-db` (`10.0.3.182`) to confirm row persistence:
   ```bash
   sudo mysql -e "USE student_db; SELECT * FROM students ORDER BY id DESC;"
   ```