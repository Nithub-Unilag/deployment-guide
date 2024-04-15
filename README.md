# deployment-guide
This repo will serve as a guide for folks who want to deploy websites or apps on NITHUB's server.

## Steps
- Ensure your ```docker-compose.yaml``` file works locally.
- Create secrets using the names highlighted below to store the server's login details (repository secrets)
```
HOST=
USERNAME=
PASSWORD=
PORT=22
```

### Setting up CI/CD Pipeline
- The pipeline configuration is located at ```.github/workflows/main.yml```
- If this your first deployment to the server, note
```
script: |
        git clone repo.git
        cd folder/
        docker-compose up -d
```

### Configure Nginx as reverse proxy
- ssh into the server and edit the Nginx configuration, located at ```nginx/config/nginx.conf```
- Add a new location block. The ```/``` after the ```location```  tag, acts as a path for your app. An example is ```https://nithubdev.unilag.edu.ng/path-to-my-app```
- At the root folder in the server, run
```
docker-compose restart
```

**Note**
- For APIs, the ```location``` block should be
```
    location /nitprofile-api {
            ..................
            proxy_pass          http://name-of-upstream/api/v1;
            proxy_read_timeout 60s;

           # Handling the URL prefix
           rewrite ^/app(/.*)$ $1 break;
            ................................
```

- In your browser or Postman, no need for the /api/v1 path in your URL. Just add the route, example
```https://nithubdev.unilag.edu.ng/nitprofile-api/health``` instead of ```https://nithubdev.unilag.edu.ng/api/v1/nitprofile-api/health```

- For non APIs, the ```location``` block should be
```
    location /nitprofile-api {
            ..................
            proxy_pass          http://name-of-upstream/;
            proxy_read_timeout 60s;
            ................................
```