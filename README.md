# Easel

## Deployment Instructions
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
