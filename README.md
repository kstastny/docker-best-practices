# Docker Best Practices

_Documentation of best practices for working with Docker according to my personal preferences. Please note that I use Windows for development, therefore the documentation is suited towards that end._


## Installation

Since Docker for Desktop comes with many issues (licensing being one of them, stability another), I prefer running Docker in WSL.

### Setup WSL distribution
To install WSL, see https://docs.microsoft.com/en-us/windows/wsl/install

1. Install Ubuntu

    `wsl --install -d Ubuntu`

1. After installation, login to the new distribution and choose username and password. The prompt should open automatically.
1. Update and upgrade packages

    `sudo apt update && sudo apt upgrade`


### Install Docker

1. Install pre-requisities

    `sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2`

1. Trust the docker repo with packages
    1. set temporary OS-specific variables
        
        `source /etc/os-release`
    1. Trust the repo
        
        `curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo apt-key add -`
    1. Add and update the repo information

        `echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list`  
        `sudo apt update`

1. Install Docker itself
    `sudo apt install docker-ce docker-ce-cli containerd.io docker-compose`

Now, you should configure the user permissions and setup docker daemon to autorun but since I tend to launch docker daemon manually, I will skip this step and just point you to [https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9](Jonathan Bowman's tutorial)


To verify the installation

1. run docker deamon `sudo dockerd`
1. run hello world container `docker run --rm hello-world`

This will print "Hello from Docker" and some informational messages if the installation is correct.


### Sources

1. [https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9](https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9)
1. [https://dev.to/_nicolas_louis_/how-to-run-docker-on-windows-without-docker-desktop-hik](https://dev.to/_nicolas_louis_/how-to-run-docker-on-windows-without-docker-desktop-hik)


## How to Change Location of Virtual Hard Drives

I have a small system disk, therefore I prefer moving the VHDs to a different harddrive to save space on the system.

1. Shutdown the WSL image. Let's assume that its name is "Ubuntu"
   
    `wsl --shutdown`

1. Export the image data

    `wsl --export Ubuntu "D:\DockerDesktop-vm-data\Ubuntu.tar"`

1. Unregister the image. Note that the ext4.vhdx will be removed automatically.

    `wsl --unregister Ubuntu`

1. Import the WSL image to different directory.

    `wsl --import Ubuntu "D:\DockerDesktop-vm-data\wsl" "D:\DockerDesktop-vm-data\Ubuntu.tar" --version 2`

1. Run the WSL distribution and verify. You may now remove the `Ubuntu.tar` file    


### Source

https://stackoverflow.com/questions/62441307/how-can-i-change-the-location-of-docker-images-when-using-docker-desktop-on-wsl2


## Basic commands


### Bash into container

`docker exec -it CONTAINER_ID bash`


## Multistage build

Multistage builds are useful for building image files that only contain the application and not any residual files used during the build. Usually the usage is such that in the first stage (docker container) the application is built, then the second stage just copies the built files and the first container is dropped.

For example, we can setup multistage build like below (example taken from https://github.com/kstastny/alexandria repo)

```docker
FROM mcr.microsoft.com/dotnet/sdk:5.0 as build

# Install node
RUN curl -sL https://deb.nodesource.com/setup_14.x | bash
RUN apt-get update && apt-get install -y nodejs
RUN npm install yarn --global

RUN yarn --version

WORKDIR /workspace
COPY . .
RUN dotnet tool restore

RUN dotnet fake build -t Publish


FROM mcr.microsoft.com/dotnet/aspnet:5.0-alpine
COPY --from=build /workspace/publish/app /app
WORKDIR /app
EXPOSE 8085
ENTRYPOINT [ "dotnet", "Alexandria.Server.dll" ]
```

In the first stage, `build`, the application is built in .net SDK image.
The second stage takes the output from the build stage and creates a container that only contains the built application.

This approach has several advantages
* Resulting docker image is smaller because it does not contain SDK and other dependencies needed for build
* no source code is left on the image
* the build is reproducible on every machine running docker. There is no need for special setup of the build server

### Sources

* https://docs.docker.com/develop/develop-images/multistage-build/


## Docker Compose

Applications usually don't run in isolation but have to communicate with other services, be it a database, message queue or other applications.

It would be very unwieldy to run everything in one container, therefore we have to separate the services into one container per service. This gives us several advantages

* services can be developed, updated and versioned separately
* we have flexibility during deployment (e.g. we can use container database for local development and managed service for production deployment)
* services can be scaled separately

We could create the containers manually and connect them using [docker network](https://docs.docker.com/network) but that would be unwieldy and error prone. That is where [docker compose](https://docs.docker.com/compose/) comes in. It allows us to define an application "stack" in one YAML file and start all the services in said stack using a single command.

### Docker Compose for Infrastructure

An example `docker-compose.yaml` file looks for https://github.com/kstastny/alexandria development looks like this.

```yaml
# format version https://docs.docker.com/compose/compose-file/compose-versioning/
version: '3.8'

# define list of services (containers) that compose our application
services:

  #identifier of database service  
  db:
    #Specify what image and version we should use; we use 10.5 due to https://jira.mariadb.org/browse/MDEV-26105
    image: mariadb:10.5
    #container restart policy https://docs.docker.com/config/containers/start-containers-automatically/
    restart: always
    #setup environment variables
    environment:
      MYSQL_ROOT_PASSWORD: mariadbexamplepassword
    #define port forwarding (host:container)
    ports:
      - 3306:3306
    #bind volume `alexandria-db` to directory `/var/lib/mysql`
    volumes:
      - alexandria-db:/var/lib/mysql

  adminer:
    image: adminer
    restart: always
    ports:
      - 8090:8080

#define volumes necessary
volumes:
  #named volume will be created automatically if it does not exist. Otherwise it is mounted
  alexandria-db:
```

This file defines the infrastructure necessary for running [Alexandria](https://github.com/kstastny/alexandria) locally. I can now easily run the infrastructure and develop against it.

To run the infrastructure, execute

> `docker-compose up --build -d`

This will compose the whole infrastructure stack and run it in the background. 

### Docker Compose for Application

If we want to run the whole application stack in one compose, for purpose of easy testing, we can just add the following service to define our docker-compose.yml.

```yaml
  alexandria:
    build:
      # build context. Either local directory or url to git repository
      context: .
      dockerfile: Dockerfile
    #https://docs.docker.com/compose/startup-order/
    depends_on:
      - "db"
    command: ["./wait-for-it.sh", "db:3306", "--", "python", "app.py"]
    ports:
      - 8080:5000
    environment:
      DB__CONNECTIONSTRING: Server=db;Port=3306;Database=alexandria;Uid=root;Pwd=${MYSQL_ROOT_PASSWORD};SslMode=none
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT}
```

This time we have used environment variable inject the database password (just for demonstration purposes).
For local run, we can store this variable in and .env file

```
MYSQL_PASSWORD=mariadbexamplepassword
```

and run the whole stack like this:
> `docker-compose --env-file .env up --build -d`

After everything is built, we can check the application logs using `docker-compose logs` command:
> `docker-compose logs -ft alexandria` 


Note that using environment variables is not ideal for passing sensitive data. Better way would be to use Docker secrets (example coming later :))


We can of course have both the file with infrastructure and application defined and choose which one to run on our leisure
> `docker-compose --env-file .env up --file docker-compose.infrastructure.yml --build -d`

### Sources

* https://docs.docker.com/compose/

