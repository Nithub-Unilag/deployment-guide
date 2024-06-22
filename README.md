**Reach out to me (repo creator) via mmaryraphaella@gmail.com before you attempt any deployment**

## Deploying your apps on the NITHUB server
Reference [this](https://gist.github.com/Mbaoma/8623408d608228a23a0d1560e84b687b) GitHub gist for more information on how to setup the CI/CD pipeline in your repository.

First, setup a CI/CD pipeline, using the reference above in your repo. Run the pipeline, ensure the container is built and running on the server.

Then add a config file named after your repo ```repo-name.conf```, to the ```nginx/config``` folder, of this ```deployment-guide``` repo.

Add your proposed URL to the list of URLs listed under the ```map $http_origin $origin_allowed``` list.

Create an upstream config for your app in the ```nginx.conf``` file and add the file's path to the ```include``` parameters.