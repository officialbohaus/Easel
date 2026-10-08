# Easel

## EC2 Deployment Instructions
```bash
#!/bin/bash
set -e

# Update packages and install system dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv postgresql postgresql-contrib nginx git curl

# Set up project directory
PROJECT_DIR="/var/www/easel"
sudo mkdir -p $PROJECT_DIR
sudo chown -R ubuntu:ubuntu $PROJECT_DIR

# Clone repository (Replace URL with repository location)
if [ ! -d "$PROJECT_DIR/.git" ]; then
    git clone https://github.com/officialbohaus/easel.git $PROJECT_DIR
fi

cd $PROJECT_DIR

# Set up Python virtual environment
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# Configure systemd service for Gunicorn
sudo bash -c 'cat > /etc/systemd/system/easel.service <<EOF
[Unit]
Description=Gunicorn instance to serve Easel Application
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/var/www/easel
Environment="PATH=/var/www/easel/venv/bin"
Environment="DATABASE_URL=postgresql://easel_user:easel_password@localhost/easel_db"
ExecStart=/var/www/easel/venv/bin/gunicorn --workers 3 --bind unix:easel.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target
EOF'

# Configure Nginx reverse proxy
sudo bash -c 'cat > /etc/nginx/sites-available/easel <<EOF
server {
    listen 80;
    server_name _;

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/easel/easel.sock;
    }

    location /static/ {
        alias /var/www/easel/static/;
    }

    location /uploads/ {
        alias /var/www/easel/uploads/;
    }
}
EOF'

# Enable Nginx site configuration
sudo ln -sf /etc/nginx/sites-available/easel /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Restart services
sudo systemctl daemon-reload
sudo systemctl start easel
sudo systemctl enable easel
sudo systemctl restart nginx

echo "Easel deployment complete!"
```

---
## Local Repo Setup
### Prerequisites:
Ensure Python 3.11+, PostgreSQL, and Git are installed.

### Clone Repository:

```bash
git clone https://github.com/your-repo/easel.git
cd easel
```
### Environment Setup:

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```
### Environment Variables Configuration:
Create a .env file in the project root:

```
FLASK_APP=app.py
FLASK_ENV=development
SECRET_KEY=your_secret_key_here
DATABASE_URL=postgresql://postgres:password@localhost:5432/easel_db
UPLOAD_FOLDER=./uploads
```
### Database Initialization:

```bash
flask db upgrade
```
### Run Application:

```bash
flask run --port=5000
```
Navigate to [http://127.0.0.1:5000](http://127.0.0.1:5000) in your web browser.
