# Traefik as Reverse Proxy for Docker

This details the setup for using Traefik https://traefik.io/ as a reverse proxy
for Docker containers hosting various web apps.

![diagram showing relationship between Traefik and Docker](./Traefik-Docker.png)

Traefik acts as a reverse proxy, listening on port 80 and redirects traffic to
the correct Docker container. Traefik itself is running in a Docker container.
Traefik listens on the Docker daemon, and detects when new containers are spun
up, and automatically provisions itself to serve traffic to that container.
When using Docker in swarm mode, muliple containers of the same image are
automatically provisioned for load balancing.

# Set up
On the production server, set up 'docker-compose.yml' and 'traefik.toml' files.

- docker-compose.yml 
  ```
  version: '2'

  services:
    traefik:
      image: traefik 
      restart: always
      command: --api --docker
      ports:
        - "80:80"     # Listen on the HTTP port
        - "8080:8080" # web admin
      networks:
        - traefik-proxy
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - ./traefik.toml:/traefik.toml
      container_name: traefik
  networks:
    traefik-proxy:
      external: true
  ```
  Line by line explanation
  - `version: '2'`: use version 2, because the production server can only use version 2 (need to upgrade docker version on production server).
  - `services:` Start the docker services
  - `traefik:` The name of the docker service
  - `image: traefik` The name of the image (if no URL given, looks in default locations on Docker Hub). This is the default Traefik image, the latest version.
  - `restart: always` If the service dies, restart. Helpful when the server is restarted, then this will automatically restart once Docker is up
  - `command: --api --docker` these are flags that are added to Traefik when it is run inside the container.
    - `--api` allows for the Web UI and the API to be active
    - `--docker` starts the Docker specific functionality
  - `ports:` Start the ports section, which ports to bind to on the host and which ports to open on the container "HOST:CONTAINER"
    - `- "80:80"` Bind to port 80 on the host, and open port 80 on the container
    - `- "8080:8080"` Bind to port 8080 on the host, and open port 8080 on the container. Used when the Web UI and API are needed.
  - `networks:` Begins the networks section
    - `- traefik-proxy` The arbitrary name of a network that the Traefik container, and all containers it cares about should be on.
  - `volumes:` Begins the volume section. This is basically a Portal that maps a folder or file on the host to a folder or file in the container. "/path/on/host:/path/in/container"
    - `- /var/run/docker.sock:/var/run/docker.sock` map the docker.sock file on the host to the path in the container. Lets Traefik listen to Docker events
    - `- ./traefik.toml:/traefik.toml` Passes the Traefik config file into the container. This file must be in the same folder (same level) as this docker-compose.yml file.
  - `container_name: traefik` Sets the name of the container.
  - `networks:` The networks available to all of the services referenced in this docker-compose.yml file. In this case, traefik-proxy should be created manually.
    - `traefik-proxy:` The network name to add attributes to
      - `external: true` This network should be visible/usable by all Docker containers.

- traefik.toml
  ```
  defaultEntryPoints = ["http"]

  [web]
  address = ":8080"

  [entryPoints]
    [entryPoints.http]
    address = ":80"
  ```
  Line by line explanation
  - `defaultEntryPoints = ["http"]` The default entry points to list to (http and https)
  - `[web]` begin the web section
  - `address = ":8080"` The address that the web UI will listen on. 
  - `[entryPoints]` begins the entryPoints section.
  - `[entryPoints.http]` begins the entryPoints section for the http entryPoint
  - `address = ":80"` http should listen to port 80

With these two files, simply start up a Docker container. Traefik will bind to
port 80 on the host and manage all incoming traffic on that port, re-routing
domains to appropriate containers.

Also, now start up some Docker containers that serve web applications. Traefik
will automatically detect them and route traffic to them as appropriate.

Web Apps in Docker containers must have the following configuration added to
their docker-compose.yml files (if they exist already) in order for Traefik to
manage them.

If a service should not be available to Traefik (the backend database
container, for example), then add a label to the service.

- db section
  ```
  labels:
    - "traefik.enable=false"
  ```

To add a service to Traefik, add the following labels:

- web app section
  ```
   networks:
     - traefik-proxy
     - default
     - internal
   expose:
     - "80"
   labels:
     - "traefik.docker.network=traefik-proxy"
     - "traefik.enable=true"
     - "traefik.port=80"
     - "traefik.backend=milio_omeka"
     - "traefik.frontend.rule=Host:cnhi-milio.nursing.virginia.edu"
  ```

And add the following networks to the docker-compose file (same level as the services line):

-  networks section
  ```
  networks:
    traefik-proxy:
      external: true
    internal:
      external: false
  ```

So the whole file would look something like this:

- docker-compose.yml (for web app)
  ```
  version: '2'

  services:
     db:
       image: mysql:5.7
       container_name: milio_mysql
       volumes:
         - ./db_data:/var/lib/mysql

         # This line is only used for the initial start up (the very first time
         # docker-compose is run or if there is no data in the 'db_data' folder
         - ./initial_sql/neatline_milio.sql:/docker-entrypoint-initdb.d/milio_initial.sql
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
         MYSQL_DATABASE: ${MYSQL_DATABASE}
         MYSQL_USER: ${MYSQL_USER}
         MYSQL_PASSWORD: ${MYSQL_PASSWORD}
       networks:
         - internal
       labels:
         - "traefik.enable=false"

     milio_omeka:
       build:
         context: .
       depends_on:
         - db
       image: milio:0.4
       container_name: milio_omeka
       #hostname: cnhi-milio.nursing.virginia.edu
       volumes:
         - ./omeka:/var/www/html/
       #ports:
         #- ${PORTS}
       restart: always
       environment:
         OMEKA_DB_HOST: ${OMEKA_DB_HOST}
         OMEKA_DB_USER: ${MYSQL_USER}
         OMEKA_DB_PASSWORD: ${MYSQL_PASSWORD}
         OMEKA_TABLE_PREFIX: ${OMEKA_TABLE_PREFIX}
       networks:
         - traefik-proxy
         - default
         - internal
       expose:
         - "80"
       labels:
         - "traefik.docker.network=traefik-proxy"
         - "traefik.enable=true"
         - "traefik.port=80"
         - "traefik.backend=milio_omeka"
         - "traefik.frontend.rule=Host:cnhi-milio.nursing.virginia.edu"
  networks:
    traefik-proxy:
      external: true
    internal:
      external: false

  ```
