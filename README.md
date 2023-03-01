# Deploy Fastapi on DigitalOcean

Deploying FastAPI Apps Over HTTPS with Traefik Proxy.

Before deployment there are some steps which needs to be followed.

##### Dockerize application

To dockerize your application make sure you have a docker installed in your local machine, Add the following file to your application directory `Dockerfile`, `.dockerignore` and `docker-compose.yml`

Add following lines to your `Dockerfile`

```
FROM tiangolo/uvicorn-gunicorn-Fastapi:python3.8
COPY ./app /app/
```

Add files name to `.dockerignore` which are not required its same like `.gitignore`

```
venv
.env
__pycache__
.DS_Store
.envrc
.gitattributes
.gitignore
.idea
.vscode
.vscode-test
```


To dockerize it run the following commands

```console
docker build -t app ./
docker run -it -p 5000:80 app
```

`app` is the name of application inside docker, in the second command `-it` it means run the application in interative terminal so we can see the application running. The port `5000:80` it means run application on port `:5000` locally on my machine and on port `:80` for docker. so it's `my_local_machine_port:inside_docker_port`, inside docker the application will run on port `:80` and will be accessible to us on `127.0.0.1:5000`.


`.docker-compose.yml` is basically a configuration file that you use to configure several docker containers that are part of the same stack, all these settings are in single place so that everything is synchronized.

Add following lines to your `docker-compose.yml` file.

```
services:

	backend:
    	build: ./
        restart: always
        ports:
        	- 80:80
```

This will run our application on port `:80` inside dokcer and also outside docker on droplet also.

To execute this file run 
```console
docker-compose up
```

##### Setup Digital Ocean Droplet

As always whenever you make a new droplet just run the update commands.

```console
apt update
apt upgrade
```

Now, Install Docker

You can also follow from official documentation `https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script`

To install docker run the following commands

```console
curl -fsSL https://get.docker.com -o get-docker.sh
sh ./get-docker.sh
```

Wait for it, it will setup everything for you.

After installation, download docker-compose on droplet by running the following commands.


```console
 curl -SL https://github.com/docker/compose/releases/download/v2.10.2/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
```

Make it executable on server
```console
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

Install the `haveged` package
```console
apt install haveged
```

Check if docker-compose is installed successfully
```console
docker-compose --version
docker info
```

The next thing is to move your project files to droplet.

Make a directory `mkdir code` and inside that directory move your project.

To send file from your local machine to remote server run

```console
rsync -a ./* username@yourserver.com:/code/yourapp
```

Run the command in /code/yourapp, on remote server

```console
docker-compose up
```

Now try to access it in your browser. Try in incognito mode if it doesn't work

```console
http://yourserver.com
```


##### Traefik for https ( TLS/SSL ) Certificates

For Traefik add the following code to the file `docker-compose.traefik.yml`

Read the documentation

`https://doc.traefik.io/traefik/user-guides/docker-compose/basic-example/`

```
services:

  traefik:
    # Use the latest v2.3.x Traefik image available
    image: traefik:v2.3
    ports:
      # Listen on port 80, default for HTTP, necessary to redirect to HTTPS
      - 80:80
      # Listen on port 443, default for HTTPS
      - 443:443
    restart: always
    labels:
      # Enable Traefik for this service, to make it available in the public network
      - traefik.enable=true
      # Define the port inside of the Docker service to use
      - traefik.http.services.traefik-dashboard.loadbalancer.server.port=8080
      # Make Traefik use this domain in HTTP
      - traefik.http.routers.traefik-dashboard-http.entrypoints=http
      - traefik.http.routers.traefik-dashboard-http.rule=Host(`yourserver.com`)
      # Use the traefik-public network (declared below)
      - traefik.docker.network=traefik-public
      # traefik-https the actual router using HTTPS
      - traefik.http.routers.traefik-dashboard-https.entrypoints=https
      - traefik.http.routers.traefik-dashboard-https.rule=Host(`yourserver.com`)
      - traefik.http.routers.traefik-dashboard-https.tls=true
      # Use the "le" (Let's Encrypt) resolver created below
      - traefik.http.routers.traefik-dashboard-https.tls.certresolver=le
      # Use the special Traefik service api@internal with the web UI/Dashboard
      - traefik.http.routers.traefik-dashboard-https.service=api@internal
      # https-redirect middleware to redirect HTTP to HTTPS
      - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
      # traefik-http set up only to use the middleware to redirect to https
      - traefik.http.routers.traefik-dashboard-http.middlewares=https-redirect
      # admin-auth middleware with HTTP Basic auth
      # Using the environment variables USERNAME and HASHED_PASSWORD
      - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
      # Enable HTTP Basic auth, using the middleware created above
      - traefik.http.routers.traefik-dashboard-https.middlewares=admin-auth
    volumes:
      # Add Docker as a mounted volume, so that Traefik can read the labels of other services
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Mount the volume to store the certificates
      - traefik-public-certificates:/certificates
    command:
      # Enable Docker in Traefik, so that it reads labels from Docker services
      - --providers.docker
      # Do not expose all Docker services, only the ones explicitly exposed
      - --providers.docker.exposedbydefault=false
      # Create an entrypoint "http" listening on port 80
      - --entrypoints.http.address=:80
      # Create an entrypoint "https" listening on port 443
      - --entrypoints.https.address=:443
      # Create the certificate resolver "le" for Let's Encrypt, uses the environment variable EMAIL
      - --certificatesresolvers.le.acme.email=admin@example.com
      # Store the Let's Encrypt certificates in the mounted volume
      - --certificatesresolvers.le.acme.storage=/certificates/acme.json
      # Use the TLS Challenge for Let's Encrypt
      - --certificatesresolvers.le.acme.tlschallenge=true
      # Enable the access log, with HTTP requests
      - --accesslog
      # Enable the Traefik log, for configurations and errors
      - --log
      # Enable the Dashboard and API
      - --api
    networks:
      # Use the public network created to be shared between Traefik and
      # any other service that needs to be publicly available with HTTPS
      - traefik-public

volumes:
  # Create a volume to store the certificates, there is a constraint to make sure
  # Traefik is always deployed to the same Docker node with the same volume containing
  # the HTTPS certificates
  traefik-public-certificates:

networks:
  # Use the previously created public network "traefik-public", shared with other
  # services that need to be publicly available via this Traefik
  traefik-public:
    external: true
```

Edit the `docker-compose.yml`, which we make before

```
services:

  backend:
    build: ./
    restart: always
    labels:
      # Enable Traefik for this specific "backend" service
      - traefik.enable=true
      # Define the port inside of the Docker service to use
      - traefik.http.services.app.loadbalancer.server.port=80
      # Make Traefik use this domain in HTTP
      - traefik.http.routers.app-http.entrypoints=http
      - traefik.http.routers.app-http.rule=Host(`yourserver.com`)
      # Use the traefik-public network (declared below)
      - traefik.docker.network=traefik-public
      # Make Traefik use this domain in HTTPS
      - traefik.http.routers.app-https.entrypoints=https
      - traefik.http.routers.app-https.rule=Host(`yourserver.com`)
      - traefik.http.routers.app-https.tls=true
      # Use the "le" (Let's Encrypt) resolver
      - traefik.http.routers.app-https.tls.certresolver=le
      # https-redirect middleware to redirect HTTP to HTTPS
      - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
      # Middleware to redirect HTTP to HTTPS
      - traefik.http.routers.app-http.middlewares=https-redirect
      - traefik.http.routers.app-https.middlewares=admin-auth
    networks:
      # Use the public network created to be shared between Traefik and
      # any other service that needs to be publicly available with HTTPS
      - traefik-public

networks:
  traefik-public:
    external: true
```

Finally add `docker-compose.override.yml`

```
services:

  backend:
    ports:
      - 80:80

networks:
  traefik-public:
    external: false
```

Move these files to server

```console
rsync -a ./* username@yourserver.com:/code/yourapp
```

Before running the file run the command
```console
docker network create traefik-public
```

Setup Environment variables

```
export USERNAME=admin
export PASSWORD=changethis
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
```

Run the new file now

```console
docker-compose -f docker-compose.traefik.yml up
```

Execute the both of the files in the backround by


```console
docker-compose -f docker-compose.traefik.yml up -d
docker-compose -f docker-compose.yml up -d
```

## Congrats 🎉
Congrats! That's a very stable way to have a production application deployed.

