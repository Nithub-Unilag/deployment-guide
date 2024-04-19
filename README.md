# Deployment-guide
This repo will serve as a guide for folks who want to deploy websites or apps on NITHUB's server.

## Overview
This README guide outlines how to set up and configure a Docker environment locally and deploy using GitHub Actions for continuous integration and deployment (CI/CD). Follow the steps below to ensure a seamless operation.

### Prerequisites
Before you begin, ensure your repo is fully dockerized and ready for deployment.

## Configuration

### Step 1: Local Docker Setup
- Ensure that your `docker-compose.yaml` file is functional on your local system.
- Add a network configuration to your Docker Compose to allow containers to communicate with each other. Below is an example of how to set up the network:
```yaml
version: '3.8'
services:
  # your service configurations go here
networks:
  default:
    name: nithub
    external: false
```

### Step 2: Secrets for Server Authentication
Create the following secrets in your repository to securely store the server's login details:
- `HOST`: The hostname or IP address of the server.
- `USERNAME`:The SSH username for the server.
- `PASSWORD`: The SSH password for the server.
- `PORT`: SSH port, typically set to 22.

## Setting up the CI/CD Pipeline
The CI/CD pipeline is configured under ```.github/workflows/main.yml```. It is crucial to review and modify this file according to your project needs.

### Pipeline Configuration Details
- The pipeline triggers on pushes to the ```main``` branch and can also be manually triggered.
- It consists of two jobs: ```build``` and ```deploy```. The ```deploy``` job depends on the successful completion of the ```build``` job.

#### Build Job
The build job prepares the environment and installs dependencies:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm install

```

#### Deploy Job
The deployment job logs into a remote server and updates the application:

- Ensure to replace ```"folder_name"``` with the actual directory name where your project is stored or will be cloned.
- Replace ```repo.git``` with your actual Git repository URL.

```yaml
  deploy:
    permissions:
      contents: none
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: login to VM and update the app
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          command_timeout: 200m
          script: |
            if [ -d "folder_name" ]; then
              echo "Directory exists. Pulling latest changes."
              cd folder_name
              git pull origin main
              docker-compose restart
            else
              echo "Directory does not exist. Cloning repository."
              git clone repo.git folder_name
              cd folder_name
              docker-compose up -d
            fi
```
## Configure Nginx as reverse proxy on server

### Step 3: SSH and Configuration File
- SSH into your server.
- Edit the Nginx configuration file located at `nginx/config/nginx.conf`.

### Step 4: Edit the Configuration
Add a new `location` block to direct traffic appropriately based on the URL path. Here is a template to guide you:
- The / after the location tag, acts as a path for your app. An example is ```https://nithubdev.unilag.edu.ng/path-to-my-app```

```nginx
upstream your-upstream {
  server                  docker-container-name:docker-port-of-service;
  keepalive               100;
  keepalive_timeout       180;
}


location /your-app {
  proxy_set_header    Host $host;
  proxy_set_header    X-Real-IP $remote_addr;
  proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header    X-Forwarded-Proto $scheme;
  proxy_set_header    Upgrade $http_upgrade;
  proxy_set_header    Connection $connection_upgrade; #keep-alive;
  proxy_http_version  1.1;
  proxy_pass          http://your-upstream;
  proxy_read_timeout 60s;

  #Handle the preflight request
  if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Allow-Origin' 'nithubdev.unilag.edu.ng';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE, PATCH' always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization,User-Agent,Keep-Alive' always;
    add_header 'Access-Control-Max-Age' 1728000;
    add_header 'Content-Type' 'text/plain charset=UTF-8';
    add_header 'Content-Length' 0;
    return 204;
  }
}
```

## Configuration Details
### For APIs 
The ```location``` block for APIs should rewrite URLs and not require the API version in the browser or tool like Postman:

```nginx
location /your-api {
  proxy_pass          http://your-upstream/api/v1;
  proxy_read_timeout 60s;

  # Handling the URL prefix to make API paths cleaner
  rewrite ^/your-api(/.*)$ /api/v1$1 break;

  # More settings...
}
```

With this, Accessing ```https://nithubdev.unilag.edu.ng/your-api/health``` will route to ```https://nithubdev.unilag.edu.ng/your-api/api/v1/health```.

### For Non APIs 
For non-API endpoints, route directly to an upstream service without additional path:

```nginx
location /some-service {
  proxy_pass          http://some-upstream-service/;
  proxy_read_timeout 60s;

  # More settings...
}
```

### Serving Static Files
To serve static files, such as images, JavaScript, or CSS, use a location block like this:

```nginx
location /static {
  root /var/www/myapp;
  expires 30d;

  # Optionally, set cache headers
  add_header Cache-Control "public";
}

```

### Step 4: Restart Nginx
Restart the Docker container running Nginx to apply the changes:
- At the root folder in the server, run
```bash
docker-compose restart
```


### example-nginx.conf
```nginx
user                nginx;
worker_processes    auto;
include             /etc/nginx/modules-enabled/*.conf;
pid                 /run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    map $http_upgrade $connection_upgrade {
      default upgrade;
      ''      close;
    }

    # Logging Settings
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Set CORS headers globally across all locations
    add_header 'Access-Control-Allow-Origin' 'https://nithubdev.unilag.edu.ng' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE, PATCH' always;
    add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    add_header 'Access-Control-Max-Age' 1728000 always;

    proxy_headers_hash_max_size 1024;
    proxy_headers_hash_bucket_size 128;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    log_format          main            '$remote_addr - $remote_user [$time_local] "$request" '
                                                '$status $body_bytes_sent "$http_referer" '
                                                '"$http_user_agent" "$http_x_forwarded_for"';

    access_log          /dev/stdout     main;
    sendfile            on;
    keepalive_timeout   160;


    ssl_certificate         /etc/letsencrypt/live/nithubdev.unilag.edu.ng/fullchain.pem; # managed by Certbot
    ssl_certificate_key     /etc/letsencrypt/live/nithubdev.unilag.edu.ng/privkey.pem; # managed by Certbot
    include                 /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam             /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    upstream nitprofile_upstream {
        server                  nitprofile-backend-refactored-app-1:4000;

        keepalive               100;
        keepalive_timeout       180;
    }

    upstream nitprofile_db_upstream {
        server                  adminer:8080;

        keepalive               100;
        keepalive_timeout       180;
    }

    server {
        listen          80;
        listen          [::]:80 ipv6only=on;

        server_name     _;
        return          301             https://$host$request_uri;
    }

    server {
        listen                  443 ssl;
        listen                  [::]:443 ipv6only=on;

        http2                   on;

        server_name             nithubdev.unilag.edu.ng;
        server_tokens           off;

        # enable HTTP Strict Transport Security (HSTS) with max-age 1 year
        # enabled by downstream service
         add_header              Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
         add_header              Referrer-Policy "no-referrer-when-downgrade" always;
         add_header              Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;

        location ^~ /.well-known/ {
            root        /usr/share/nginx/html;
            allow       all;
        }

        ## FOR APIs
        location /nitprofile-api {
            proxy_set_header    Host $host;
            proxy_set_header    X-Real-IP $remote_addr;
            proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header    X-Forwarded-Proto $scheme;
            proxy_set_header    Upgrade $http_upgrade;
            proxy_set_header    Connection $connection_upgrade; #keep-alive;
            proxy_http_version  1.1;
            proxy_pass          http://nitprofile_upstream/api/v1;
            proxy_read_timeout 60s;

           # Handling the URL prefix
           rewrite ^/app(/.*)$ $1 break;

            # UNCOMMENT TO FIX CORS ERROR
            #Handle the preflight request
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' 'nithubdev.unilag.edu.ng';
                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE, PATCH' always;
                add_header 'Access-Control-Allow-Credentials' 'true' always;
                add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization,User-Agent,Keep-Alive' always;
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
        }
      }

        ## ADD A NEW LOCATION BLOCK BELOW like this
        location /your-app {
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header    X-Forwarded-Proto $scheme;
          proxy_set_header    Upgrade $http_upgrade;
          proxy_set_header    Connection $connection_upgrade; #keep-alive;
          proxy_http_version  1.1;
          proxy_pass          http://your-upstream;
          proxy_read_timeout 60s;

          #Handle the preflight request
          if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' 'nithubdev.unilag.edu.ng';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE, PATCH' always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization,User-Agent,Keep-Alive' always;
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
          }
        }
        
        ## If you're working with static files
        location /static {
          root /var/www/myapp;
          expires 30d;

          # Optionally, set cache headers
          add_header Cache-Control "public";
        }
    }
}
```

**Note**
- Reference [this](https://docs.nginx.com/nginx/admin-guide/web-server/serving-static-content/) when serving static files.