# Dockerize Django with Nginx Certbot ( SSL Certificate )

Django is a high-level, open-source web framework for Python that encourages rapid development and clean, pragmatic design. A framework made by group of perfectionist for perfectionists with deadlines. In this blog we'll explore how to dockerize a Django application with Nginx and secure it with a free Let's Encrypt SSL certificate. By creating portable, lightweight containers, a containerization technology known as Docker makes it simpler to deploy and manage programs. Nginx is a well-known web server and reverse proxy server, and Let's Encrypt is a free, automatic, and open Certificate Authority (CA) that provides SSL/TLS certificates for protecting websites. This project also covers setting up celery, elasticsearch, flower and kibana


#### 💻 Prerequisites

- Basic understanind of Docker, Django, Nginx
- A domain pointing to your server
- A VM Instance or a server with an open 443 and 80 port
- A VM instance with Ubuntu or Debian Based System

#### What we will achieve 

- A ssl secured web app written in Django pointed to a domain
- Automated ssl certificate renewal 
- Celery workers in the backend
- Elasticsearch and Postgres setup
- Redis setup
- Celery worker monitor tool setup 
- Kibana to view elasticseasrch results

#### 💻 Setup

For referance we will use the following github repo | [Referance Repo](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt)

Clone this repo in your local system to anaylyze

```bash
git clone https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt
```

==blog_api== is the python django application, a basic blog api which we will use as our referance project. Along with this project we have celery, elasticsearch, kibana covered as well.

In the project root we have a dockerfile for the django application

```Dockerfile
# Debian based 
FROM python:3.11.0-bullseye

ENV PYTHONUNBUFFERED 1

# Install required package for postgres and django
RUN apt-get update && apt-get install -y libpq-dev gcc python3-dev musl-dev build-essential

# We used pipenv for our project
RUN pip install pipenv

COPY Pipfile Pipfile
COPY Pipfile.lock Pipfile.lock

RUN pipenv install

COPY ./blogs_api ./blogs_api

WORKDIR blogs_api
```

replace 'blogs_api' with your project name.

==docker-compose.yaml== contains dev compose file, suitable for dev environment. You must have a .dev.env file in the project root containing dev environment variables. View .sample.env in the ref repo

### 🪜 Steps

### SSH Into your VM

Managing a web server can be a daunting task, especially for those who are new to server administration. One of the most popular server operating systems is Ubuntu, and using SSH to connect to an Ubuntu-based server is a common practice. After successfully creating your droplet / vm machine, allow ssh to your vm. You can use either your username password or ssh key to get access in your instance.

```bash
ssh root@your_instance_ip_address
```

### Connect your domain and server

To point your domain to the DigitalOcean Droplet, follow these steps

- Log in to your domain registrar's control panel.
- Locate the DNS management settings.
 - Add a new "A" record with the following details:
 - Name: @ (or your desired subdomain)
 - TTL: 3600 (or the recommended value)
 - Type: A
- Value: your_droplet_ip
- Save the changes.

It may take some time for the DNS changes to propagate.

### Download and install Docker 🐋

Download docker in your system and enable docker service, so even if you restart your vm it will automatically turn on your docker containers

( Example for Ubuntu/Debian Based Systems )

The following is taken from official docker doc page

```bash
# App Environment 
sudo su

mkdir /home/app -p

cd /home/app

apt-get update
```

Remove old versions of docker and install latest ( Steps taken from official docker page, [View More](https://docs.docker.com/engine/install/ubuntu/))

```bash
# Uninstall Old Versions
apt-get remove docker docker-engine docker.io containerd runc
apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

Add Docker Repo
```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update apt and install docker
```bash
# Install docker engine
apt-get update -y
# Install docker
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# Install docker-compose package
apt-get install -y docker-compose-plugin docker-compose
```

For other os [visit docker official documentation](https://docs.docker.com/engine/install/)


### Setup Github repo 😺

To get our codebase from github repo we need to generate a ssh deploy key
and put the deploy key public key to Github repo deploy key section

```bash
ssh-keygen -t ed25519 -C "GitHub Deploy Key"
```

Cat the public and copy and paste it in the github deploy keys section

Github Repo > Settings > Deploy Keys

```bash
cat ~/.ssh/id_ed25519.pub
```

The following command will cat the public key

```bash
ssh-ed25519 AAAAC3NzaD23ZDI1NTE5ABSFGKoH0ASXx2ua/++wZgCUSDGsg6VmPc/ys7vNSDGsd2D6 GitHub Deploy Key
```

Setting deploy key

![alt Image](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt/blob/main/images/image1.png)

Put public key `ssh-ed25519 AAAAC3NzaD23ZDI1NTE5ABSFGKoH0ASXx2ua/++wZgCUSDGsg6VmPc/ys7vNSDGsd2D6 GitHub Deploy Key` In the `Key` section

![alt Image](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt/blob/main/images/image2.png)


### Pull code from github repo

```bash
git init
```

Add your repo origin git url

```bash
git remote add origin <OriginURL>
```

Here for example I am pulling from main
```bash
git pull origin main
```
### Setup environment variable 💲

Create .env file where your environment variables will be stored
```bash
nano .env
```

```dotenv
DOMAIN=<DOMAIN>
ACME_DEFAULT_EMAIL=<EMAIL>
ENV=PROD
SECRET_KEY=<SECRET_KEY>
ENCRYPTION_KEY=<ENCRYPTION_KEY>
POSTGRES_DB=POSTGRES_DB
POSTGRES_USER=POSTGRES_USER
POSTGRES_PASSWORD=POSTGRES_PASSWORD
DATABASE_URL=postgresql://<POSTGRES_USER>:<POSTGRES_PASSWORD>@db:5432/<POSTGRES_DB>
REDIS_URL=redis://redis:6379/1
CELERY_BROKER_URL=redis://redis:6379/0
AWS_STORAGE_BUCKET_NAME=<AWS_STORAGE_BUCKET_NAME>
AWS_S3_REGION_NAME=<AWS_S3_REGION_NAME>
AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
AWS_SECRET_ACCESS_KEY=AWS_SECRET_ACCESS_KEY
AWS_SECRET_ACCESS_KEY_ID=AWS_SECRET_ACCESS_KEY_ID
AWS_S3_ENDPOINT_URL=AWS_S3_ENDPOINT_URL
AWS_LOCATION=static
DEBUG=False
EMAIL_HOST=EMAIL_HOST
EMAIL_HOST_USER=EMAIL_HOST_USER
EMAIL_HOST_PASSWORD=EMAIL_HOST_PASSWORD
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
DEFAULT_FROM_EMAIL=DEFAULT_FROM_EMAIL
ACCESS_TOKEN_LIFETIME=ACCESS_TOKEN_LIFETIME
REFRESH_TOKEN_LIFETIME=REFRESH_TOKEN_LIFETIME
ELASTICSEARCH_HOST=http://elasticsearch:9200
```

### Configure and run docker 🐋

Build docker containers, we will use the docker-compose.prod.yaml file as it has the production level configuration
```bash
docker-compose -f docker-compose.prod.yaml build 
```

Run certbot command to get initial ssl certificate
```bash
docker-compose -f docker-compose.prod.yaml run -rm certbot /opt/certify.sh
```

Restart containers

```bash
docker-compose -f docker-compose.prod.yaml down
```

```bash
docker-compose -f docker-compose.prod.yaml up -d
```

### Crontab to renew ssl certificate 📜

The actions mentioned above will produce the first certificate for our project.
The certificate will only be valid for three months, 
therefore you must run the renew command earlier than that.

Create a bash script
```bash
nano rewnew.sh
```

This bash script will renew certificate
```bash
#!/bin/sh
set -e

cd /home/app/

/usr/local/bin/docker-compose -f docker-compose.prod.yaml run --rm certbot certbot renew
```

Give execution permission
```bash
chmod +x renew.sh
```

Open crontab
```bash
crontab -e
```

Add the following in the crontab
```
0 0 * * 6 sh /home/app/renew.sh
```

### Automated deploy using github actions 🎬

Create an SSH Key

```bash
ssh-keygen -t rsa -b 4096
```

Copy content of the key file

```bash
cat /.ssh/id_rsa
```

Add public key to the authorized key file

```bash
cat /.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Create a bash script to deploy
```basah
nano deploy.sh
```

This bash script will renew certificate
```bash
#!/bin/sh
set -e

cd /home/app/

git stash

git pull origin main --force

/usr/local/bin/docker-compose -f docker-compose.prod.yaml up --build -d
```

Give execution permission
```bash
chmod +x deploy.sh
```


### [Optional] Setup Github Actions

- Go to “Settings > Secrets > Actions”
- 
![alt Image](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt/blob/main/images/image3.png)


Open the Actions tab and set secrets

![alt Image](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt/blob/main/images/image4.png)


Create the following secrets:

SSH_PRIVATE_KEY: content of the private key file

SSH_USER: user to access the server (root)

SSH_HOST: hostname/ip-address of your server

WORK_DIR: path to the directory containing the repository ( For this /home/app)

MAIN_BRANCH: name of the main branch (mostly main)


![alt Image](https://github.com/khan-asfi-reza/django-docker-nginx-letsencrypt/blob/main/images/image5.png)


In your repo create

`.github > workflows > deploy.yaml`

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  run_pull:
    name: run pull
    runs-on: ubuntu-latest

    steps:
    - name: install ssh keys
      # check this thread to understand why its needed:
      # https://stackoverflow.com/a/70447517
      run: |
        install -m 600 -D /dev/null ~/.ssh/id_rsa
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        ssh-keyscan -H ${{ secrets.SSH_HOST }} > ~/.ssh/known_hosts
    - name: connect and pull
      run: ssh ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} "cd ${{ secrets.WORK_DIR }} && git checkout ${{ secrets.MAIN_BRANCH }} && /home/app/deploy.sh"
    - name: cleanup
      run: rm -rf ~/.ssh
```

The following github actions will automatically deploy the code and restart the container whenever you push in main branch

