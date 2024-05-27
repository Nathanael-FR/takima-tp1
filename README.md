# Discover Docker

## Database

### Basics

```
docker build -t nathanaelfr/mypostgresdb:v1 .
```

### Adminer

A lightweight database management tool to interact with our PostgreSQL instance via a web interface.

### Network

`docker network create app-network` creates a user-defined bridge network, allowing containers to communicate by name.

#### Why should we run the container with a flag -e to give the environment variables?

 Using the -e flag allows you to customize environment variables at runtime. This means we can use the same image across different environments (development, testing, production) with different configurations without modifying the image itself.

In our case :

```
docker run --name my-postgres-container \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -v ./data:/var/lib/postgresql/data \
  --network app-network \
  -d nathanaelfr/mypostgresdb:v2.0
```

### Init database

`COPY ./init.sql /docker-entrypoint-initdb.d/` copies the SQL scripts within the **/init.sql** directory to /docker-entrypoint-initdb.d/. Tgus allows PostgreSQL to initialize the database with our schema and data automatically.

### Persist data

#### Why do we need a volume to be attached to our postgres container?

```
-v ./data:/var/lib/postgresql/data 
```

It ensures that PostgreSQL data is stored on the host filesystem, so it persists even if the container is removed, by copying the content of data directory inside the container into the /data directory.


## Backend API

### Multistage build

#### 1-2 Why do we need a multistage build? 

A multistage build in Docker is used to create smaller, more efficient Docker images by separating the build environment from the runtime environment. This approach helps in keeping the final image lightweight and secure, containing only the necessary dependencies to run the application.

#### 1-2 Explain each step of this dockerfile.

* **Build stage**

```
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests
```

1. Base Image for Building:

`FROM maven:3.8.6-amazoncorretto-17 AS myapp-build`: Uses a Maven image with Amazon Corretto JDK 17 to compile the Java application. This image includes all necessary tools to build the application.

2. Environment Variable:

`ENV MYAPP_HOME /opt/myapp`: Sets an environment variable MYAPP_HOME to define the application directory.

3. Working Directory:

`WORKDIR $MYAPP_HOME`: Sets the working directory inside the container to /opt/myapp.

4. Copy Files:
`COPY pom.xml .`: Copies the pom.xml file to the working directory in the container.
`COPY src ./src`: Copies the src directory to the working directory in the container.

5. Build the Application:

`RUN mvn package -DskipTests`: Runs the Maven package command to compile the application and package it into a JAR file, skipping the tests to speed up the build process.

* **Runtime Stage**

1. Base Image for Running:

`FROM amazoncorretto:17`: Uses a smaller Amazon Corretto JDK 17 image suitable for running the Java application.

2. Environment Variable:

`ENV MYAPP_HOME /opt/myapp`: Sets the same environment variable MYAPP_HOME.

3. Working Directory:

`WORKDIR $MYAPP_HOME`: Sets the working directory inside the container to /opt/myapp.

4. Copy Built Artifact:

`COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar`: Copies the JAR file built in the previous stage into the runtime image. The `--from=myapp-build` directive specifies that the source files come from the myapp-build stage, that's why we used an alias at this stage.

5. Run the Application:

`ENTRYPOINT java -jar myapp.jar`: Sets the entry point for the container to run the Java application using the JAR file.

### Backend API

*Adjust the configuration in simple-api/src/main/resources/application.yml*

```
url: jdbc:postgresql://${POSTGRE_DB:mypostgresdb}:${POSTGRE_PORT:5432}/{POSTGRE_DB:db}
```

Here we use environemental variables as recommended above. If not provided when running the countainer, it will use the default values on right side.

## HTTP Server

### Reverse Proxy

A reverse proxy server is an intermediate server that sits between client devices and backend servers, forwarding client requests to the appropriate backend server and returning the server's response to the client.

```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://${BACKEND_HOST}:8080/
ProxyPassReverse / http://${BACKEND_HOST}:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

* `<VirtualHost *:80>` : Indique que cette configuration s'applique à toutes les adresses IP du serveur sur le port 80.
*  `ProxyPass / http://${BACKEND_HOST}:8080/`Indique à Apache de rediriger toutes les requêtes reçues à la racine (/) vers le serveur backend spécifié dans les variables d'environnement.


## Link application

### Why is docker-compose so important?

A tool for defining and running multi-container Docker applications. All container definitions and configurations are stored in a single docker-compose.yml file, making it easy to manage and share the setup. Start, stop, and rebuild all your containers with single commands. 

### 1-3 Document docker-compose most important commands. 

`docker-compose up/stop/down/build/logs/ps`

In order : 

* Builds, creates, starts, and attaches to containers for a service.
* Stops running containers without removing them.
* Stops and removes containers, networks, volumes, and images created by up.
* Builds or rebuilds services defined in docker-compose.yml.
* Lists all the containers created by docker-compose up.

#### Some useful parameters :

`docker-compose up --force-recreate` : Forcer la recréation des conteneurs même s'ils existent déjà.
`docker-compose --env-file .env up` :  Charge les variables d'environnement à partir d'un fichier spécifié.
`docker-compose restart <service_name>` : Redémarre un service spécifique.


### 1-4 Document your docker-compose file.

```
version: '3.7' # Specifies the version of the Docker Compose file format.

services:
    api:
        container_name: api # Specifies the name of the container.
        image: nathanaelfr/simpleapistudent:v4.0 # Specifies the image to use for the container.
        networks:
            - app-network # Specifies the network to which the container should be connected, here 'app-network'.
        depends_on:
            - database # Specifies the service on which the container depends, here the postgres database.
        environment:    # Specifies the environment variables to set in the container.  
            - DB_HOST=mypostgresdb
            - DB_PORT=5432
            - DB_NAME=db
            - DB_USER=usr
            - DB_PWD=pwd
    database:
        container_name: mypostgresdb 
        image: nathanaelfr/tp1:v2.0
        environment:
            POSTGRES_DB: db
            POSTGRES_USER: usr
            POSTGRES_PASSWORD: pwd
        volumes:
            - db-data:/var/lib/postgresql/data # Specifies the volumes to mount in the container.
        networks:
            - app-network

    server:
        container_name: server
        image: nathanaelfr/proxy:v2.0
        environment:
            - BACKEND_HOST=api 
        ports:
            - "8085:80" # Specifies the ports to expose on the host machine.
        networks:
            - app-network
        depends_on:
            - api

networks: # Specifies the networks to create, here 'app-network'.
    app-network: 

volumes: # Specifies the volumes to create, here 'db-data'.
    db-data: 
```

## Publish 

As we can see in the docker-compose file the image used for the postgresdb doesn't have a proper name so we can change the name. It duplicates the image with another name.

```
PS C:\Users\natha> docker tag nathanaelfr/tp1:v2.0 nathanaelfr/postgresdb:v2.0
```



link to the Docker Hub repo : https://hub.docker.com/u/nathanaelfr

Example documented image for the postgresdb : https://hub.docker.com/r/nathanaelfr/postgresdb 

### Why do we put our images into an online repo?

* Version Control: Using tags, different versions of images can be managed and referenced, ensuring that everyone uses the correct version
* CI/CD Integration: Online repositories can be integrated into continuous integration and continuous deployment pipelines, automating the build and deployment process.
