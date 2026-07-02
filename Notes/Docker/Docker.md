### Container
- A way to package application with all the necessary dependencies and configuration
- Portable artifact, easily shared and moved around
- makes development and deployment easier

Where do containers live?
Images live in container repositories
Containers run on Docker hosts
- Public repositories - Docker Hub
- private repositories - self hosted by companies


##### How Containers help Application Development

Without container we need to install the related software dependencies  and has different configurations. so its difficult to setup and cant run different versions at a same time

With containers we no need to install any dependencies. everything is packaged with in the container.

- its has own isolated environment
- packaged with all needed configuration
- one command to install the app
- can run same app with 2 different version at same time


##### How containers help Application Deployment
Before Container
Assume we have a jar file, a DB and some set of configurations. 
developer send them to operations.
so configuration on the server is needed. everything need to be installed in the server(os) and configured. might cause
- dependency version conflicts
- textual guide of deployment
- misunderstanding -- developers missed to tell important dependency/config to operations or operations has missed a configuration.
- this can lead to back and forth communication to deploy the server

After Container
- Developers and Operations work together to package the application in a container.
- Since it is already encapsulated in single environment -- No environmental configuration needed on server - except Docker running.
- run a docker command to pull the container image somewhere in repository and run it



## Container
- An image is made of multiple read-only layers.
- Mostly Linux Base image(alpine:3.10), because small in size
- Application image on top

```
postgres:10.10  Layer - Application image
	  ↓
   SomeLayer
       ↓
   Some Layer
       ↓
    alpine:3.10     → Layer - linux base image
```

Docker Container
- actually start the application
- container environment is created
- when it is running is called a container

Docker Image 
- the actual package
- artifact, that can be moved around -not running


### Docker Vs Virtual Machine

Docker Virtualizes the Application Layer of the OS.
VM virtualizes the complete OS i.e. both application layer and OS kernal

Docker size much smaller as they have one or few images of particular software(some MB)
VM size is in GB

Docker containers start and run much fast

VM of any OS can run on any OS Host. but docker cannot
the linux based image might not be compactible with window host. (mostly the older versions)  --Historically
can run using docker toolbox it abstracts the kernal.

Now they run as - Docker Desktop uses a lightweight Linux VM internally.


### Container vs Image
- container is a running environment for image
- Image --> postgres, redis, mongo
- container has a port binded to it. to talk with the application inside container
- Container has a virtual file system which is different form local OS file system.


### Docker commands

List all running containers
```
docker ps
```

List all containers which are running and idle
```
docker ps -a
```

Run an image
```
docker run <image-name>
```

run a image in detached mode
```
docker run -d <image-name>
```

stop a docker running container
```
docker stop <container-name>
```

start a specific image which already exists
```
docker start <container-name>
```

To run a image of a particular version
```
docker run <image-name>:<version>
docker run <image-name>:<version2>
```

>Run command will check if the image is present is locally if not. it will perform docker pull and then it will perform docker run

In order to check the logs of a particular container
```
dockers logs <container-id>
			or
docker logs <container-name>
```

Naming a container
```
docker run --name <name> <image-name>:<version>
```

To get a terminal for a running container
```
docker exec -it <container-id> /bin/bash
			or
docker exec -t <container-name> /bin/bash
```




### Container Port vs Host Port
- multiple containers can run on your host machine
- laptop has only certain ports available
- will get a conflict when same port is used on host
- its absolutely fin. to have same port for container as long as they binded to different host ports
- an container can be binded to a particular port by using the below command
```
docker run -p <host_port>:<container_port> <image_name>
```


### Environment Variables

Environment Variables are configuration values injected into a container at runtime.

To Set Env Variables
```
docker run \
-e DB_URL=jdbc:postgresql://postgres:5432/devsync \
-e DB_USER=postgres \
-e DB_PASSWORD=secret \
my-app
```

Verify Env Variables
```
docker exec -it container_id env
```

Output
```
DB_URL=...
DB_USER=...
DB_PASSWORD=...
```


### Volumes
By default container storage is temporary
```
docker run postgres
```
insert data and stop container
remove container
```
docker rm postgres
```
Data is gone

Problem
- Containers are disposable
Database data must survive

Soln: Volumes

Docker stores data outside the container filesystem.
```
docker volume create postgres-data
```

Run:
```
docker run \
-v postgres-data:/var/lib/postgresql/data \
postgres
```

Types of storage
##### Anonymous Volume
```
-v /data
```
docker generate volume name

##### Named Volume
```
-v postgres-data:/data
```

Bind mount

```
-v C:/projects/devsync:/app
```
or
```
-v ./src:/app/src
```

Maps local folder directly


### Networks
Containers cannot magically find each other.
Docker Networks allow communication.

Create a network
```
docker network create <network-name>
```

Run Image:
```
docker run \
--network <network-name> \
--name postgres \
postgres
```

```
docker run \
--network devsync-network \
--name app \
my-app
```

Now app can reach database using postgres not localhost

Inspect
```
docker network ls
docker network inspect <network-name>
```

Remove docker network
```
docker network rum <network-name / naetowrk id>
```

### Dockerfile

A Dockerfile is a blueprint for building images
```
Dockerfile
      ↓
docker build
      ↓
Image
      ↓
docker run
      ↓
Container
```

DockerFile:
```
FROM eclipse-temurin:21-jdk

WORKDIR /app

COPY target/devsync.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java","-jar","app.jar"]
```

Build image:
```
docker build -t devsync .
```

Run:
```
docker run -p 8080:8080 devsync
```

Common Commands

>FROM \<base-image\>

it tells the base image for our application image
```
FROM openjdk:21
```

>WORKDIR

Working directory
```
WORKDIR /app
```

> COPY

Copy files
```
COPY . .
```

> RUN

Execute command while building image
```
RUN mvn clean package
```

> EXPOSE

Document port
```
EXPOSE 8080
```

> ENTRYPOINT

Container startup command
```
ENTRYPOINT ["java","-jar","app.jar"]
```


### Docker Compose
Real applications require multiple containers.

running all manually is painful

We create one file to run entire stack
```
version:'3'
services:
  postgres:
    image: postgres:17

  app:
    image: devsync
```

Exmaple
```
services:
  postgres:
    image: postgres:17
    container_name: postgres
    environment:
      POSTGRES_DB: devsync
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
  app:
    build: .
    container_name: devsync
    ports:
      - "8080:8080"
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/devsync
      DB_USER: postgres
      DB_PASSWORD: secret
    depends_on:
      - postgres
volumes:
  postgres-data:
```

Start everything 
```
docker compose up
```
Stop everything
```
docker compose down
```