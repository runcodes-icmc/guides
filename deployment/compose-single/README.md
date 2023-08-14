# Docker Compose (Single Machine) Deployment Guide

This guide will walk you through the process of deploying the run.codes project to a server, thinking about a
production environment. For local development, we recommend to just use the included Docker Compose setup on
most of the repositories.

## First Steps

### Requirements

We strongly recommend that your deployment target is a Linux machine, as it is the only one we tested and the one used by the Docker images (if you don't there might be extra setup steps for Docker due to virtualization).

In order to deploy the project using Docker Compose, you will need to have both **Docker** and **Docker Compose** installed
to your deployment target. To do so, we recommend you follow the official guides for [Docker](https://docs.docker.com/engine/install/)
for your distribution. Docker Compose is usually included by default (as it is now a Docker plugin).

And, that's it! You are ready to go. All the other dependencies will be handled by the Docker images.

### Other Considerations

Later into the guide, we will need some resources that you might need to acquire before starting the deployment process. Those are:

- An e-mail server (SMTP) to send e-mails from the application.
  - You can use any standard e-mail server, as an example, we will use a Gmail account, which will share the setup steps if you use a Google Workspace account.

- A domain name to use for the application.
  - During this deployment, you'll only need a single domain name, but if you plan to later switch to a multi-machine deployment, it might be useful to have domains (most likely subdomains) for both the file storage service and the server.

- Storage
  - There will be lots of container going up and down by the application (as it is used to run the submissions), and you'll also need to have images for each supported language's container and to store the files uploaded by the users. We recommend you have at least **50GB** of storage available for the application, but it will depend on your usage.

## Deployment

### Step 0: Dependencies

Make sure to have all the requirements installed and ready to go before starting the deployment process. Check [this section](#requirements) for more information.

### Step 1: Deployment Files

The first thing needed is to download the deployment files. Those are the files that will be used to deploy the application, and they will be used in the following steps.

#### Step 1.1: Create the deployment directory

Let's create a deployment directory for our application. This directory will contain most of the files we will need to deploy the application, alongside some other files that will be generated during the deployment process like the database files and storage pool.

We'll use for this example the directory `/opt/run.codes`, but you can use any directory you want (just be sure to replace it on all the following commands too).

```bash
mkdir -p /opt/run.codes
```

#### Step 1.2: Download the deployment files

Inside this repository, you'll find a directory called `guides/deployment/compose-single` (where this guide is housed). This directory contains all the files we will need to deploy the application using Docker Compose. They consist of five files:

- `docker-compose.yml`: The main Docker Compose file, which will be used to deploy the application.
- `.env.example`: An example `.env` file, which will be used to configure the application.
- `pull-compiler-images.sh`: A small utility script to pull the compiler images for the supported languages.
- `s3-config.json`: The SeaweedFS' S3 configuration file (for credential setup).
- `Caddyfile`: The Caddy configuration file (for the reverse proxy).

To download them, you can use the following commands (assuming you have `curl` installed):

```bash
# Download the docker-compose.yml file
curl -L -o /opt/run.codes/docker-compose.yml https://raw.githubusercontent.com/run-codes/guides/main/deployment/compose-single/docker-compose.yml

# Download the .env.example file
curl -L -o /opt/run.codes/.env.example https://raw.githubusercontent.com/run-codes/guides/main/deployment/compose-single/.env.example

# Download the pull-compiler-images.sh file
curl -L -o /opt/run.codes/pull-compiler-images.sh https://raw.githubusercontent.com/run-codes/guides/main/deployment/compose-single/pull-compiler-images.sh

# Download the s3-config.json file
curl -L -o /opt/run.codes/s3-config.json https://raw.githubusercontent.com/run-codes/guides/main/deployment/compose-single/s3-config.json

# Download the Caddyfile file
curl -L -o /opt/run.codes/Caddyfile https://raw.githubusercontent.com/run-codes/guides/main/deployment/compose-single/Caddyfile
```

Make sure to also make the `pull-compiler-images.sh` file executable:

```bash
chmod +x /opt/run.codes/pull-compiler-images.sh
```

### Step 2: Configuration

Now that we have our files ready, we'll need to adjust some configurations before we can deploy the application. We have included
a sensible set of defaults in both the `docker-compose.yml` and `.env.example` files, but you might want to change some of them. Now we are
going to tackle the core configurations, but you might take a closer look at the other configurations too.

We'll need to create some credentials for our application to use. We strongly recommend using some random source to generate the credentials, like the following command:

```bash
# Generate a random string of 64 hex digits
openssl rand -hex 32
```

If you don't have `openssl` installed, you can use any other random string generator, like [this one](https://www.random.org/strings/?num=1&len=64&digits=on&upperalpha=on&loweralpha=on&unique=on&format=html&rnd=new).

#### Step 2.1: The .env file

Most of the configurations will be set in the `.env` file. It is a simple text file with the following format:

```env
KEY=VALUE
```

Where `KEY` is the name of the configuration and `VALUE` is the value of the configuration. You can change the value of any configuration by changing the `VALUE` part of the line.

You'll need to have the `.env` file in the same directory as the `docker-compose.yml` file, so let's create a copy of the example file to the correct location:

```bash
# Create the .env file from the example file
cp /opt/run.codes/.env.example /opt/run.codes/.env
```

Now, let's take a look at the configurations we'll need to change (other entries can be left as they are):

##### Step 2.1.1: Domains

- `RC_APP_DOMAIN`: This is the main domain of your application, which will be used for redirects,
  certificate issuing and **routing** (you'll need to access the application using this domain, or
  else the proxy won't route the request). If you have a domain configured, use it here. If you don't
  have a domain configured, you can use the IP address of your server here (but you'll need to
  access the application using the IP address, or else the proxy won't route the request).
  **Do not include the protocol**

- `RC_FILES_DOMAIN`: This is the files domain (used to access uploads). If you have a separate
  domain/subdomain for that purpose, use it here. If you don't have a separate domain/subdomain,
  make sure to use the same value as the `RC_APP_DOMAIN` configuration with a trailing `/seaweed`,
  as this information is used by the server to send the correct URLs to the client. Later on we'll
  adjust our proxy to serve the files from this path, if needed. **Do not include the protocol**

##### Step 2.1.2: Server

- `RC_CONTACT_EMAIL`: This is the email address that will be used to register the SSL certificate
  for your application. It will also be referenced on some parts of the application and will be used
  for critical notifications. Make sure to use a valid email address here.

- `RC_SECURITY_SALT`: This is the salt used to persist hashed information (like passwords) into the
  database (sadly fixed, as CakePHP 2 used this way by default). You should use a random string generated
  as described above.

- `RC_SECURITY_CIPHER_SEED`: This is the seed used to encrypt/decrypt information. Not sure if it is
  used anywhere, but you should use a random string generated as described above. **It must be a
  integer**. You can generate it with the command: `od -N 8 -t uL -An /dev/urandom | tr -d " "`. You
  can also use [this website](https://www.random.org/strings/?num=1&len=18&digits=on&unique=off&format=html&rnd=new).

##### Step 2.1.3: Email

- `RC_SMTP_HOST`: This is the SMTP host that will be used to send emails.
- `RC_SMTP_PORT`: This is the SMTP port that will be used to send emails.
- `RC_SMTP_USER`: This is the SMTP username that will be used to send emails.
- `RC_SMTP_PASS`: This is the SMTP password that will be used to send emails.
- `RM_EMAIL_SENDER_ADDRESS`: This is the email address that will be used to send emails.
- `RM_EMAIL_SENDER_NAME`: This is the name that will be used to send emails.

##### Step 2.1.4: Database

- `RC_DB_PASSWORD`: This is the password that will be applied to the two default users: `postgres`
  and `runcodes`. Again, you should use a random string generated as described above. (By default, 
  the database is not exposed, being kept inside the Docker network).

##### Step 2.1.5: Storage

- `RC_SEAWEED_KEY`: This is the key that will be used to access the SeaweedFS server. You could use
  a random string generated as described above (not as important as the secret one).
- `RC_SEAWEED_SECRET`: This is the secret that will be used to access the SeaweedFS server. You should
  use a random string generated as described above.

### Step 2.2: Caddyfile

Now that we have our application configured, we'll need to configure our proxy. We'll use Caddy as our
proxy, as it is a simple and powerful web server with automatic HTTPS support. The default configuration
provided in the `Caddyfile` file should be enough for most cases, but you might want to look at it to
see your options.

**Note:** If you are using a separate domain/subdomain for the files, you'll need to uncomment
the `Hosting (files domain)` section.

### Step 2.3: SeaweedFS S3 Config

We need to adjust the SeaweedFS S3 configuration to match our setup. We'll need to change the `accessKey` and `secretKey` to match the values we set in the `.env` file.

### Step 3: Deployment

Now that you have everything configured, you can deploy the application. Let's walk through the steps:

#### Step 3.1: Pulling the images

The compiler service requires that the images are pulled before the application is deployed. You can do that by running the following command:

```bash
# Pull the images
/opt/run.codes/pull-compiler-images.sh
```

This is step can be a bit slow, as it will download the images from the registry (still faster then
compiling them). It is only needed when you are deploying the application for the first time or when
you are updating the images.

#### Step 3.2: Deploying the application

To deploy the application with docker compose, you can run the following command:

```bash
# Ensure you are in the correct directory
cd /opt/run.codes

# Deploy the application
docker-compose up -d
```

This will start the application in the background. You can check the logs with the following command:

```bash
# Check the logs
docker-compose logs -f
```

You can stop the application with the following command:

```bash
# Stop the application
docker-compose down
```

And that's it! You should now be able to access the application using the domain you configured.

### Step 4: Setting up some jobs (optional, yet recommended)

Due to some unsolved bugs, it is recommended to setup some jobs to run periodically. You can do that by adding the following lines to your crontab (`crontab -e` command, as root):

```bash
# Restart the compiler service at 02:00 AM
0 2 * * * bash -c "cd /opt/run.codes && docker-compose restart rcc"

# Restart the server service at 02:00 AM
0 2 * * * bash -c "cd /opt/run.codes && docker-compose restart app"
```

### Step 5: Backups (optional)

This isn't a very well defined step, as it will wildly vary depending on your setup and necessities. But,
if it is important for you to have backups, you should probably look into it. The main directories you
should backup are: `/opt/run.codes/seaweed` (uploads) and `/opt/run.codes/postgres` (database).
