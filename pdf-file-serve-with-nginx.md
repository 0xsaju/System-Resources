# Certificate Server Setup Documentation with nginx

## Overview
This document outlines the setup process for a certificate serving system using Nginx. The system consists of two servers:
- A host server (192.168.1.10) that stores and serves the PDF certificates
- A proxy server that handles SSL termination and forwards requests to the host server

## Directory Structure
```
Host Server (192.168.1.10):
/opt/sqa_training/
└── cert/
    └── YYYYMM/
        ├── 01.pdf
        ├── 02.pdf
        └── ...

/etc/nginx/
├── sites-available/
│   └── sqa.conf
└── sites-enabled/
    └── sqa.conf -> ../sites-available/sqa.conf
```

## Setup Process

### 1. Host Server Setup (192.168.1.10)

#### 1.1 Create Required Directories
```bash
# Create certificate directory
sudo mkdir -p /opt/sqa_training/cert

# Create sample batch directory (e.g., for October 2024)
sudo mkdir -p /opt/sqa_training/cert/202410
```

#### 1.2 Set Directory Permissions
```bash
# Set ownership
sudo chown -R www-data:www-data /opt/sqa_training
sudo chmod -R 755 /opt/sqa_training
```

#### 1.3 Configure Nginx on Host Server
Create configuration file:
```bash
sudo nano /etc/nginx/sites-available/sqa.conf
```

Add the following configuration:
```nginx
server {
    listen 80;
    server_name certificate.example.com;
    
    # Access logs
    access_log /var/log/nginx/certificates_access.log;
    error_log /var/log/nginx/certificates_error.log;
    
    # Root directory for certificates
    root /opt/sqa_training;

    # Certificate serving location
    location /cert/ {
        # Disable directory listing
        autoindex off;
        
        # Allow PDF files
        location ~ \.pdf$ {
            # Set proper content type
            types { application/pdf pdf; }
            
            # Set content disposition to display in browser
            add_header Content-Disposition inline;
            
            # Cache control
            add_header Cache-Control "public, no-transform";
            
            # Try serving the file, return 404 if not found
            try_files $uri =404;
        }
    }

    # Deny access to all other locations
    location / {
        deny all;
    }
}
```

#### 1.4 Enable Configuration
```bash
sudo ln -s /etc/nginx/sites-available/sqa.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 2. Proxy Server Setup

#### 2.1 Configure Nginx on Proxy Server
Create configuration file:
```bash
sudo nano /etc/nginx/sites-available/sqa.conf
```

Add the following configuration:
```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name certificate.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS Server
server {
    listen 443 ssl;
    server_name certificate.example.com;
    
    # SSL Configuration
    include snippets/ssl_cert.conf;
    include snippets/ssl_params.conf;
    
    # Access logs
    access_log /var/log/nginx/proxy_certificates_access.log;
    error_log /var/log/nginx/proxy_certificates_error.log;
    
    # Proxy settings
    location / {
        proxy_pass http://192.168.1.10;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeout settings
        proxy_connect_timeout 60;
        proxy_send_timeout 60;
        proxy_read_timeout 60;
        
        # Buffer settings
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
    }
}
```

#### 2.2 Enable Configuration
```bash
sudo ln -s /etc/nginx/sites-available/sqa.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Certificate Management

### Adding New Certificates
1. Create the batch directory (if it doesn't exist):
```bash
sudo mkdir -p /opt/sqa_training/cert/YYYYMM
```

2. Copy certificate files:
```bash
sudo cp /path/to/certificate.pdf /opt/sqa_training/cert/YYYYMM/XX.pdf
```

3. Set proper permissions:
```bash
sudo chown www-data:www-data /opt/sqa_training/cert/YYYYMM/XX.pdf
sudo chmod 644 /opt/sqa_training/cert/YYYYMM/XX.pdf
```

### Certificate URL Format
Certificates are accessible via:
```
https://certificate.example.com/cert/YYYYMM/XX.pdf
```
Where:
- `YYYYMM`: Year and month of the batch (e.g., 202410 for October 2024)
- `XX`: Certificate number (e.g., 01, 02, etc.)

## Maintenance

### Log Files
- Host Server:
  - Access Log: `/var/log/nginx/certificates_access.log`
  - Error Log: `/var/log/nginx/certificates_error.log`
- Proxy Server:
  - Access Log: `/var/log/nginx/proxy_certificates_access.log`
  - Error Log: `/var/log/nginx/proxy_certificates_error.log`

### Common Commands
```bash
# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx

# Reload Nginx configuration
sudo systemctl reload nginx

# Check Nginx status
sudo systemctl status nginx

# View logs
sudo tail -f /var/log/nginx/certificates_error.log
```

## Security Considerations
1. Only PDF files are served
2. Directory listing is disabled
3. Access to other locations is denied
4. SSL/TLS encryption is enabled
5. Proper file permissions are set

## Troubleshooting
1. Check Nginx error logs for specific error messages
2. Verify file permissions and ownership
3. Ensure SSL certificates are valid
4. Check network connectivity between proxy and host servers
5. Verify DNS resolution for the domain

## Backup
Regular backups of the following are recommended:
- Certificate files in `/opt/sqa_training/cert/`
- Nginx configuration files
- SSL certificates and keys
