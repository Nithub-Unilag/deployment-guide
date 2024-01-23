- Contents of *file-name.yaml*
```bash
name: Deployment Name

# Configure this workflow to run every time a change is pushed to the ```deploy``` branch or a pull request is opened to the ```hotfix``` or ```production-branch-name``` branch
on:
  push:
    branches:
      - "dev-branch-name"
  pull_request:
    branches:
      - "staging-branch-name"
      - "production-branch-name"

env:
  FORCE_COLOR: 3 #enables color coding in the CI output, to help with  easy identification of errors or warnings or success messages
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  dev-branch-name: #pipeline steps for a particular branch
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      matrix:
        python-version: ["3.12"]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install system dependencies
      run: |
        sudo apt-get update
        sudo apt-get -y install libpq-dev gcc

    - name: Install pipenv and dependencies
      run: |
        pip install -U pipenv
        pipenv install --system --deploy --ignore-pipfile

  staging-branch-name:
    runs-on: ubuntu-latest
    #Sets the permissions granted to the GITHUB_TOKEN for the actions in this job.
    permissions:
      contents: read
      packages: write
    if: github.ref == 'refs/heads/staging-branch-name' || github.event.pull_request.base.ref == 'staging-branch-name'
    needs: ['branch-name']
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      #Uses the docker/login-action action to log in to the Container registry registry using the account and password that will publish the packages. Once published, the packages are scoped to the account defined here.
      - name: Log in to the Container registry
        uses: docker/login-action@65b78e6e13532edd9afa3aa52ac7964289d1a9c1
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build, tag, and push image to GHCR
        id: build-image
        uses: docker/build-push-action@f2a1d5e99d037542a71f64918e516c093c6f3fc4
        with:
          context: .
          push: true
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

    #deploy app to server
    production-branch-name:
    runs-on: ubuntu-latest
    #Sets the permissions granted to the GITHUB_TOKEN for the actions in this job.
    permissions:
      contents: read
      packages: write
    if: github.ref == 'refs/heads/production-branch-name' || github.event.pull_request.base.ref == 'production-branch-name'
    needs: ['staging-branch-name']
    steps:
      - name: Deploy to On-Prem Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          password: ${{ secrets.SERVER_PASSWORD }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            docker stop django-app || true
            docker rm django-app || true
            docker run -d --name django-app -p 8000:8000 ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

```