---
layout: post
title: "Dockerizing a Node.js + MongoDB Web-App"
author: "Krithik Vaidya"
tags: [containers, docker, docker-compose, node, express, mongodb]
image: Dockerizing-Application/moby-logo.png
---

**Note**: This is a write-up for a recruitment task of the Web Enthusiasts' Club, NITK.
The problem statement is given below.  

<br>

## Problem Statement: 
- The objective is to dockerize a Node.js application and wrap the setup.
- Install and set up the Node.js application. [This](https://github.com/Eslamunto/ProductsApp) is the code for the app and [this](https://github.com/Eslamunto/ProductsApp) is the tutorial on how to set it up. (No need to fully know how the app works or how to develop it, only the basics given in the tutorial.). You can use any other web application also instead of the one provided, under the condition that it uses an SQL database. In case you’re using a different app, please discuss it with the mentors first.
- Dockerize the app and make sure the database of the app is made persistent.
- Wrap the docker setup using docker-compose for easy management (Bonus task).  

<br>

# Detailed Explanation
<br>
## What is Docker?

Before understanding what Docker is, we need to know what a Container is -
*Containerization* is the process of packaging up an application with all the parts it needs to run - library functions, dependencies, and shipping it all out in a single package known as a **Container**. This ensures that the application can be quickly spun up for use in any other Linux based system, without needing to specifically install any required dependencies and dealing with other kinds of software conflicts. In other words, the containerization helps deploy applications as portable, self-sufficient containers. It ensures that the app always runs in the same environment, and that there aren't any inconsistencies on different systems. Makes collabarative development easier

**Docker** is basically an open-source containerization technology that enables the creation and use of Linux containers. Docker containers are lightweight because they don’t need the extra load of a hypervisor (which is a hardware level virtualization technique that allows multiple guest operating systems (OS) to run on a single host system at the same time. Multiple instances of a variety of operating systems may share the virtualized hardware resources, which contrasts with the operating-system-level virtualization that containerization uses, where all instances must share a single kernel), but run directly within the host machine’s kernel. This means you can run more containers on a given hardware combination than if you were using virtual machines. You can even run Docker containers within host machines that are actually virtual machines! Docker is written in Go and takes advantage of several features of the Linux kernel to deliver its functionality.

In a way, Docker similar to a virtual machine. However, rather than creating a whole virtual operating system on top of our host operating system, Docker allows applications to use the same Linux kernel as the system that they're running on and only requires applications be shipped with things not already running on the host computer. This gives a significant performance boost and reduces the size of the containerized application, as compared to if it were to be run on a virtual machine.

One of the most important features Docker offers is it’s instant startup time. A Docker container can be started within milliseconds, as opposed to waiting minutes for a virtual machine to boot.  
<br>

# Our application
<br>

## Setup of the application to be containerized
Initially, we setup a Node.js app by following [this](https://codeburst.io/writing-a-crud-app-with-node-js-and-mongodb-e0827cbbdafb) tutorial.
However, we will setup our database a little differently.
Replace 
```
let dev_db_url = 'mongodb://someuser:abcd1234@ds123619.mlab.com:23619/productstutorial';
let mongoDB = process.env.MONGODB_URI || dev_db_url;
mongoose.connect(mongoDB);
mongoose.Promise = global.Promise;
let db = mongoose.connection;
db.on('error', console.error.bind(console, 'MongoDB connection error:'));
```
in the *app.js* file with
```
mongoose
  .connect(
    'mongodb://mongo:27017/docker-node-mongo',
    { useNewUrlParser: true }
  )
  .then(() => console.log('MongoDB Connected'))
  .catch(err => console.log(err));
```
This is because we are not using a remote database; we will be using a database that runs locally within its own container named *mongo*. The name we give our database within mongo is *docker-node-mongo*.

Also, open up the *package.json* file and add another key-value pair to the "scripts" dictionary -
```
"start": "node app.js"
```
So that the *package.json* file looks like this:
```
...
"scripts": {
    "start": "node app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
},
...
```
(The "test" parameter can be removed) .
This just indicates what should happen when the *npm start* command is run (more on this later). 

## Creating the files required for Dockerizing this application
First, we create a Dockerfile to setup our a container for our Node.js app and install all its dependencies. We will now run through what we'll be writing in our Dockerfile:  

1. ```FROM node:latest```  
This specifies the base image on which we wish to build on. It'll look in the Docker Registry (https://hub.docker.com) for a matching image and download it to our system.

2. ```WORKDIR /usr/src/app```  
The WORKDIR instruction sets the working directory for any RUN, CMD, ENTRYPOINT, COPY and ADD instructions that follow it in the Dockerfile. If the WORKDIR doesn’t exist, it will be created.

3. ```COPY package*.json ./```  
This copies all files having the name package*.json (where * is the wildcard character, indicates that there can be any character(s) in that position) to our *WORKDIR* (which is reference by ./ since we already set our WORKDIR in the previous command).

4. ```RUN npm install```  
This runs the command *npm install* which looks in our *package.json* file (generated when we installed packages through *npm* and gave the *--save* parameter), and installs all the dependencies mentioned in it.

5. ```COPY . .```  
Copies all files and folders from the directory in which our Dockerfile resides to the WORKDIR of the container (excluding the files and folders mentioned in the *.dockerignore* file).

6. ```EXPOSE 3000```  
This statement is just for documenting which port(s) are intended to be published by our container (here, 3000), but does not actually map or open any ports.

7. ```CMD [ "npm", "start" ]```  
This runs the *npm start* command in our container, which looks in our *package.json* for the "scripts" key (similar to the below)
```
"scripts": {
    "start": "node app.js"
},
...
```
and runs whatever command is specified by the "start" key.

These commands specify how the images for the containers running our Node.js app will look like. However, we still haven't dealt with the other container in our application - the container for our mongo database.

## Creating the docker-compose.yml file

Since our app consists of two different components (the main Node.js app and our MongoDB database), we will need two different Docker containers for each of them. To make these two components work together like a single unit, we use create a Docker Compose file(**docker-compose.yml**). Compose is a tool for defining and running multi-container Docker applications. 

Using Compose is basically a three-step process:

1. Define your app’s environment with a Dockerfile so it can be reproduced anywhere.

2. Define the services that make up your app in docker-compose.yml so they can be run together in an isolated environment.

3. Run 
```
sudo docker-compose up --build
```
and Compose starts and runs our entire app.

We will be using docker-compose to spin up a container for our *mongo* container, as well as create a kind of 'composite container', combining our previously created container to this one. Here is a line-by-line explanation of our *docker-compose.yml*:

1. ```version: '3'```  
Specifies the docker compose file format. Version 3 is currently the latest version.

2. ```services: ```  
Defines the services (one for our main *app* and another for *mongo*) that make up our app in so they can be run together in an isolated environment. We define two services - *app* and *mongo*

3. ```app:```  
The name of our first service

4. ```container_name: docker-node-mongo```  
The name of our first container

5. ```restart: always```  
Specifies that the container always restarts whenever requested. Other possible values are:  
restart: "no"  
restart: on-failure  
restart: unless-stopped  

6. ```build: .```  
Configuration options that are applied at build time. It basically tells Docker to look for a Dockerfile in the current directory, and use it to build the image.

7. Ports  
```
ports:    
     '1234:3000'
```
Maps port *3000* of our container to port *1234* of the host system.

8. 
```
depends_on:
      mongo
```
Expresses dependency between our *app* service and another service named *mongo* (defined next in the same file). This means that the *mongo* service will be started before our app service starts.

9. ```mongo:```  
The beginning of the definition of our second service, named *mongo*

10. ```container_name: docker-node-mongo```  
The name of our second container

11. ```image: mongo```  
Use the *mongo* image(if not cached locally, will pull from Docker Hub) as the base image for our *mongo* container.

12. 
```
ports:
      '27017:27017'
```
Maps port *27017* of our container to port *27017* of the host system.

13. 
```
volumes:
      /data/db
```
Containers are ephemeral and once a container is removed, it's data is lost.
The fundamental thing that we are trying to do is to separate out the container lifecycle from the data. Ideally we want to keep these separate so that the data generated is not destroyed or tied to the container lifecycle and can thus be reused. This is done using *volumes* in Docker  
 
As per the official documentation, there are 2 ways in which you can manage data in Docker:  
- Data volumes  
- Data volume containers  

We will use data volumes.  

The explanation for what exactly has been done with the above command is given in the **Quick Explanation** section above

## The .dockerignore file
When we build an image from a Dockerfile using the docker build command the docker-daemon will create a context. That context contains everything in the directory you executed the command in.
The .dockerignore file allows us to exclude files from the context like a .gitignore file allow you to exclude files from your git repository.
It helps to make build faster and lighter by excluding from the context big files or repository that are not used in the build.
[Source](https://stackoverflow.com/questions/31789770/what-files-are-the-dockerignore-work-on)

So we add
```
node_modules/
npm-debug.log
```
to the .dockerignore file to prevent our local modules and debug logs from being copied onto our Docker image and possibly overwriting modules installed within our image.  

<br>

## How to use
1. Clone my [solution](https://github.com/krithikvaidya/WEC-Docker-Task) repository to your local machine

2. Make sure you have [Docker](https://docs.docker.com/install/) installed on your local machine

3. Install [Docker Compose](https://docs.docker.com/compose/install/)

4. Open a terminal, cd to this Docker-Task folder

5. Run 
```
sudo docker-compose up --build
```

6. If everything goes fine, your console will finally output -
```
docker-node-mongo | MongoDB Connected
```
If you get a MongoNetworkError, restarting docker usually fixes it. If you get an error saying Mongo is unable to listen at port 27017, 
```
sudo systemctl restart docker
```

7. Test the application by making a GET request to  
```
localhost:1234/products/test 
```
(by using [Postman](https://www.getpostman.com/) or just simply visiting the link on your browser). You should be see the following text -  
```
Greetings from the Test controller!
```
verifying that our app was properly setup.

8. You can now make **GET, POST, PUT, DELETE** requests to this
running server using **Postman** and the procedure followed in [this article](https://codeburst.io/writing-a-crud-app-with-node-js-and-mongodb-e0827cbbdafb)

9. If you wish to run the MongoDB shell for the app's database, open up a new terminal window, cd to this Docker-Task folder,
and making sure that the container is running, type
```
mongo
```
An interactive mongo shell should open up.

(Alternatively, we could have added the *--detached* parameter while running our *docker-compose up* command, which would have caused our container to run in the background, letting us type 
commands in the same terminal window. So we could have run the *mongo* command without opening another terminal window)

- Additionaly, we can also verify that two containers have been created by running
```
sudo docker container ls
```
which will display the details of currently running containers, that looks similar to this:
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                      NAMES
7c0e58ae932a        docker_app          "docker-entrypoint.s…"   15 seconds ago      Up 13 seconds       0.0.0.0:1234->3000/tcp     docker-node-mongo
540fe6382bf7        mongo               "docker-entrypoint.s…"   15 seconds ago      Up 14 seconds       0.0.0.0:27017->27017/tcp   mongo
```

## Data Persistence 
Note the very last configuration command written under our *mongo* service:
```
    volumes:
        - /data/db
```
This maps the data storage location of our container to the directory */data/db* (relative to the 
directory in which this *docker-compose.yml* file resides)
This ensures that whatever the data volume the container creates persists inside the */data/db* directory even
when the container has stopped running. So, the next time we start up the container, the same volume data will still be restored.

To see exactly which directory of the container was mapped to the */data/db* directory, run the command
```
sudo docker inspect mongo
```
Where *mongo* is the name of the container in which our *MongoDB* is running(as specified in the *container-name* attribute of the *mongo* service in our **docker-compose.yml**)

In the JSON-format output, look for a key named *"Mounts"*. You should find
```
{
    "Type": "volume",
    "Name": "0ec01aec488629f740132dc655ad8474c96980b1d0c8d2208aecb30bb2384598",
    "Source": "/var/lib/docker/volumes/0ec01aec488629f740132dc655ad8474c96980b1d0c8d2208aecb30bb2384598/_data",
    "Destination": "/data/db",
    ...
    ...
    ...
    ...
}
```

This tells you that when you mounted a volume (*/data/db*), it has created a folder /var/lib/… for you, which is where it puts all the files, etc that you would have created in that volume.  

<br>


# References
[https://docs.docker.com/get-started/](https://docs.docker.com/get-started/)
[https://docs.docker.com/engine/docker-overview/](https://docs.docker.com/engine/docker-overview/)
[https://docs.docker.com/compose/gettingstarted/](https://docs.docker.com/compose/gettingstarted/)
[https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)
[https://opensource.com/resources/what-docker](https://opensource.com/resources/what-docker)
[https://www.youtube.com/watch?v=YFl2mCHdv24](https://www.youtube.com/watch?v=YFl2mCHdv24)
[https://www.youtube.com/watch?v=Kyx2PsuwomE](https://www.youtube.com/watch?v=Kyx2PsuwomE)
[https://www.youtube.com/watch?v=hP77Rua1E0c](https://www.youtube.com/watch?v=hP77Rua1E0c)
[https://www.codementor.io/blog/docker-technology-5x1kilcbow](https://www.codementor.io/blog/docker-technology-5x1kilcbow)
