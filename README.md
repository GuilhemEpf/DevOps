# TP DevOps Peron Guilhem

## TP Part 1- Docker

1.  ### Database 

     #### a. Create image and run container
    ```sh
    FROM postgres:14.1-alpine
    ENV POSTGRES_DB=db \\
       POSTGRES_USER=usr \\
       POSTGRES_PASSWORD=pwd
    

We create a Dockerfile which import a postgreSQL image on a specific
version. We use environment variable to define the name of the DB along
with its username and password.
    
```sh
    docker build -t my-postgres .
    docker run -d \--name my-postgres -p 5432:5432 my-postgres
```

We then create the image indicating the path, in our case the dot is
justified by the fact that we place ourselves in the right folder so it
searchs by default for 'Dockerfile'-named fil. Then we create and start
the container at the same time with the run command.

```sh
    docker network create app-network
```

We then create a network to link all of our future container to make
their communication easier, allowing to hide their port from the
exterior.

```sh
    docker run -p 8090:8080 \--net=app-network \--name=adminer -d adminer
```

We use a container adminer, which change the interface to have a well
organized one instead of a rudimentary one

     #### b. Create table



```ssh
    CREATE TABLE public.departments

    (

    id SERIAL PRIMARY KEY,

    name VARCHAR(20) NOT NULL

    );

    CREATE TABLE public.students

    (

    id SERIAL PRIMARY KEY,

    department_id INT NOT NULL REFERENCES departments (id),

    first_name VARCHAR(20) NOT NULL,

    last_name VARCHAR(20) NOT NULL

    );
```

We create two tables and import it into our database with the command:

```ssh
COPY ./sql-scripts/\* /docker-entrypoint-initdb.d/
```
c.  Volumes

Finaly, we can store the data on our local disk in case where the
container is deleted:
```ssh
-v /my/own/datadir:/var/lib/postgresql/data
```
2.  ### Backend 

    a.  Basics
```ssh
public class Main {

public static void main(String\[\] args) {

System.out.println(\"Hello World!\");

}

}
```
We create a main.java with this code which aim to return a message to
our console.

We create a Dockerfile
```ssh
FROM openjdk:23-slim-bookworm

COPY ./Main.class /usr/src/myapp/Main.class

WORKDIR /usr/src/myapp

CMD \[\"java\", \"Main\"\]
```
Then build our images and run the container similarly to earlier. It
then shows the "Hello World" message on our console.

b.  Multistaging

We add in our Dockerfile:
```ssh
FROM eclipse-temurin:17-jdk-alpine

\# Build Main.java with JDK

\# TODO : in next steps (not now)

FROM eclipse-temurin:17-jre-alpine

\# Copy resource from previous stage

COPY \--from=0 /usr/src/Main.class .

\# Run java code with the JRE

\# TODO : in next steps (not now)
```

1-2 Why do we need a multistage build? And explain each step of this
dockerfile.

the goal of the multistaging is to lighten the weight of the image by
separating the steps of building, optimizing security and handling of
dependencies.

For now we don't touch it and get the springboot application. We add a
simple Greetingcontroller Class to Display a "Hello World" to the
localhost.

Now we can change our multistaging:
```ssh
\# Build

FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

ENV MYAPP_HOME /opt/myapp

WORKDIR \$MYAPP_HOME

COPY pom.xml .

COPY src ./src

RUN mvn package -DskipTests

\# Run

FROM amazoncorretto:17

ENV MYAPP_HOME /opt/myapp

WORKDIR \$MYAPP_HOME

COPY \--from=myapp-build \$MYAPP_HOME/target/\*.jar
\$MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

Here we compile a java application with maven and then run it in a
lightweight container using the amazoncorreto JRE.

We take the new simple-api-student-main and put the dockerfile in.
change the url in the application:

url:  jdbc:postgresql://database:5432/db

We repeat the process used to the database image and container for our
backend. Without forgetting to link to our network.

We can now access our database directly on our localhost.

### Frontend 

We create very basic interface with a index.html
```html
\<!DOCTYPE html\>

\<html lang=\"en\"\>

\<head\>

    \<meta charset=\"UTF-8\"\>

    \<meta name=\"viewport\" content=\"width=device-width,
initial-scale=1.0\"\>

    \<title\>Welcome\</title\>

\</head\>

\<body\>

    \<h1\>Welcome to my website!\</h1\>

\</body\>

\</html\>
```

In a new Dockerfile :

FROM httpd:2.4
```ssh
COPY ./index.html/ /usr/local/apa2/htdocs/
```
Import the tool httpd and copy the index in the container.

Then again we create our image and container with docker build and
docker run.

### Reverse Proxy 

To configure the proxy we add to our config this part:
```ssh
\<VirtualHost \*:80\>

ProxyPreserveHost On

ProxyPass / http://backend:8080/

ProxyPassReverse / http://backend:8080/

\</VirtualHost\>
```
LoadModule proxy_module modules/mod_proxy.so

LoadModule proxy_http_module modules/mod_proxy_http.so

It defines a reverse proxy to the backend while listening the port
80:80. This allow to redirect the get coming from the http front to the
backend api.
```ssh
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```
This line copies the new conf into the container conf.

### Docker Compose

Now after installing Docker-compose we create a docker-compose.yml where
the goal will be to spare ourselves all the rebuilding of images and
running of container.

So basically, we will gather everything we've done since the beginning
in a file that will do it all at once without us having to do it
manually many times:
```ssh
version: \'3.7\'

services:

    backend:

        container_name: backend

        build: .\\API\\simple-api-student-main\\

        environment:

            DB_host: database

            DB_port: 5432

            DB_name: db

        ports:

            - \"8080:8080\"

        networks:

            - app-network

        depends_on:

            - database

    database:

        container_name: database

        build: .\\db\\

        environment:

            POSTGRES_DB: db

            POSTGRES_USER: usr

            POSTGRES_PASSWORD: pwd

        volumes:

            - postgresdata:/var/lib/postgresql/data

        networks:

            - app-network

    httpd:

        container_name: httpd

        build: .\\Frontend http\\

        environment:

            BACKEND_host: backend

        ports:

            - \"80:80\"

        networks:

            - app-network

        depends_on:

            - backend

networks:

    app-network:

volumes:

    postgresdata:
```
so we create three services, one for each layer of our server. In each
we insert the necessary information:

-   Name of the container we will create

-   Link to the file where the Dockerfile is to build the image with the
    correct JRE.

-   Port if needed to communicate

-   Dependency on other services to make the link inside the network
    between the services.

-   The network it belongs to

-   And the volume for the database because we have stored the
    information inside.

-   Environment variables for the services to be able to get the
    information needed for exemple the backend service need to access
    the database with the information about database 'host, port,name.

### Publish

In order to no longer store our image locally, but rather where it can
be accessed by others, we will publish them on Docker hub.

We will use these two command to publish them:

```ssh
docker tag my-database guilhemperon/my-database:1.0

docker push guilhemperon/my-database:1.0
```

## TP Part 2- Github Actions

### Build and test your Application

After downloading maven library we run this command:

mvn clean verify

which is supposed to clean the previous builds and rebuild them while
also running the different tests.

2-1 What are testcontainers?

Test containers are java library providing to the docker container
things that can be run such as lighten databases for example. The whole
point about them is to make sure the tests run are isolated, in order to
have an easier time improve the tests or even doing more in a clean and
organized way.

In there, there are unit test and component tests where "Unit tests"
test code step by step isolating each part/layer, and "Component tests"
are supposed to make sure the integration is well made by testing that
each part interacts with each other without a hitch.

### Build and test your Application
```ssh
  \# Trigger the workflow on push to main and develop branches

  push:

    branches:

      - main

      - develop

  pull_request:

jobs:

  test-backend:

    runs-on: ubuntu-22.04

    steps:

      \# Checkout yourr GitHub code using actions/checkout@v2.5.0

      - uses: actions/checkout@v2.5.0

      \# Set up JDK 17 using actions/setup-java@v3

      - name: Set up JDK 17

        uses: actions/setup-java@v3

        with:

          distribution: \'temurin\'

          java-version: \'17\'

      \# Build and test your app with Maven

      - name: Build and test with Maven

        run: mvn clean verify \--file
/home/gperon/DevOps-1/API/simple-api-student-main/pom.xml  
```

Now that we made sure the integration was okay with all of our tests
passed on the github actions we can proceed to the deployment.

### First steps into the CD World

We add a new jobs to build and push our docker images:
```ssh
    >   \# define job to build and publish docker image
    >
    >   build-and-push-docker-image:
    >
    >     needs: test-backend
    >
    >     \# run only when code is compiling and tests are passing
    >
    >     runs-on: ubuntu-22.04
    >
    >  
    >
    >     \# steps to perform in job
    >
    >     steps:
    >
    >       - name: Checkout code
    >
    >         uses: actions/checkout@v2.5.0
    >
    >       - name: Login to DockerHub
    >
    >         run: docker login -u \${{ secrets.USER_DOCKERHUB }} -p \${{
    > secrets.PASSWORD_DOCKERHUB }}
    >
    >      
    >
    >  
    >
    >       - name: Build image and push backend
    >
    >         uses: docker/build-push-action@v3
    >
    >         with:
    >
    >           context: ./API/simple-api-student-main
    >
    >           tags:
    >  \${{secrets.USER_DOCKERHUB}}/tp-devops-simple-api:latest
    >
    >           push: \${{ github.ref == \'refs/heads/main\' }}
    >
    >       - name: Build image and push database
    >
    >         uses: docker/build-push-action@v3
    >
    >         with:
    >
    >           context: ./db
    >
    >           tags:  \${{secrets.USER_DOCKERHUB}}/tp-devops-db:latest
    >
    >           push: \${{ github.ref == \'refs/heads/main\' }}
    >
    >  
    >
    >       - name: Build image and push httpd
    >
    >         uses: docker/build-push-action@v3
    >
    >         with:
    >
    >           context: ./Frontend http
    >
    >           tags:  \${{secrets.USER_DOCKERHUB}}/tp-devops-httpd:latest
    >
    >           push: \${{ github.ref == \'refs/heads/main\' }}
    >
    >  
```
so we connect to dockerhub thanks to secret key we created with github
that contains all information need to do it. Then we build images and
push them on it.

### Setup Quality Gate

Now we will use sonarcloud to analyze our codes and give feedback on its
quality, security etc.
```ssh
  test-backend:

    runs-on: ubuntu-22.04

    steps:

      \# Checkout yourr GitHub code using actions/checkout@v2.5.0

      - uses: actions/checkout@v2.5.0

      \# Set up JDK 17 using actions/setup-java@v3

      - name: Set up JDK 17

        uses: actions/setup-java@v3

        with:

          distribution: \'temurin\'

          java-version: \'17\'

      \# Build and test your app with Maven

      - name: Build and test with Maven

        run: mvn -B verify sonar:sonar -Dsonar.projectKey=projectblabla
-Dsonar.organization=devops-sonarcloudtp
-Dsonar.host.url=https://sonarcloud.io -Dsonar.login=\${{
secrets.SONAR_TOKEN }}  \--file ./API/simple-api-student-main/pom.xml
```
 

here we changed the maven part to link it with our newly created project
on Sonarcloud, again we used secret key to allow registration on the
account.

## TP Part 3- Ansible

### Inventories 

Now the next step in our role as a DevOps would be to automatize
everything we've done so that we no longer have to do it again manually

A host server was prepared so we just need to configure in the setup
file the corresponding information.
```ssh
all:

 vars:

   ansible_user: centos

   ansible_ssh_private_key_file:
/home/gperon/DevOps-1/my-project/ansible/inventories/ssh-privatekey

 children:

   prod:

     hosts: guilhem.peron.takima.cloud
```
We can test our connection to the server thanks to the ping command that
test it for us.
```ssh
-   ansible all -i inventories/setup.yml -m ping
```
then we set up our OS distribution and remove the apache httpd:
```ssh
-   ansible all -i inventories/setup.yml -m setup -a
    \"filter=ansible_distribution\*\"

-   ansible all -i inventories/setup.yml -m yum -a \"name=httpd
    state=absent\" \--become
```
### Playbook

The playbook will serve as the place where we run the different roles we
create, each roles being here to display a layer of our server
(database, backend, proxy, and implementation of docker and ansible).

But first we test its use with the ping command
```ssh
\- hosts: all

gather_facts: false

become: true

tasks:

\- name: Test connection

ping:
```
```ssh
ansible-playbook -i inventories/setup.yml playbook.yml
```
Now that we got familiar with the use of the playbook let's configure
the roles that will be launched one by one by the playbook.

ansible-galaxy init roles/docker

this will create a role and inside the task folder we will add our code
to execute.

#### Network role: 
```ssh
\-\--

\# tasks file for roles/docker

\# Install Docker

\- name: Create network

  docker_network:

    name: app-network

  vars:

    ansible_python_interpreter: /usr/bin/python3
```
here we create the network which is very important for security and
communication with every layer.

#### Docker role:

In this role we make sure all configuration related to docker and
ansible are well installed.
```ssh
\-\--

\# tasks file for roles/docker

\# Install Docker

  - name: Install device-mapper-persistent-data

    yum:

      name: device-mapper-persistent-data

      state: latest

  - name: Install lvm2

    yum:

      name: lvm2

      state: latest

  - name: add repo docker

    command:

      cmd: sudo yum-config-manager
\--add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker

    yum:

      name: docker-ce

      state: present

  - name: Install python3

    yum:

      name: python3

      state: present

  - name: Install docker with Python 3

    pip:

      name: docker

      executable: pip3

    vars:

      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running

    service: name=docker state=started

    tags: docker
```
#### database role:

here we first pull our image from our already created Dockerfile and
then run the database while naming it, setting its image along with the
state and link it to the network.
```ssh
\- name: pull db image

  docker_image:

    name: guilhemperon/tp-devops-db  

    tag: latest

    source: pull

  vars:

    ansible_python_interpreter: /usr/bin/python3

\- name: Run DB

  docker_container:

    name: database

    image:  guilhemperon/tp-devops-db

    state: started

    networks:

      - name: app-network

  vars:

    ansible_python_interpreter: /usr/bin/python3
```
#### backend role:

we do the same as with the database.
```ssh
\- name: pull API image

  docker_image:

    name: guilhemperon/tp-devops-simple-api

    tag: latest

    source: pull

  vars:

    ansible_python_interpreter: /usr/bin/python3

\- name: Run APP

  docker_container:

    name: backend

    image: guilhemperon/tp-devops-simple-api

    state: started

    networks:

      - name: app-network

  vars:

    ansible_python_interpreter: /usr/bin/python3

```

#### proxy role:

again we do the same process but this time we add a port in the run part
in order for the server to be visible by communicating on a port.
```ssh
\-\--

\# tasks file for roles/docker

\# Install Docker

\- name: pull proxy

  docker_image:

    name: guilhemperon/tp-devops-httpd

    tag: latest

    source: pull

  vars:

    ansible_python_interpreter: /usr/bin/python3

\- name: Run proxy

  docker_container:

    name: httpd

    image: guilhemperon/tp-devops-httpd

    ports:

      - \"80:80\"

    networks:

      - name: app-network

   

  vars:

    ansible_python_interpreter: /usr/bin/python3
```

#### final playbook running everything:

here we run everything in the correct order.
```ssh
\- hosts: all

  gather_facts: false

  become: true

  roles:

    - docker

    - Network

    - launch-database

    - Launch-app  

    - Proxy  
```
### Continuous deployment

We make another .yml file in the github workflow (we could have made
just another job in the main.yml file too) and make it run only if the
main has been successfully run, hence deploying the application on the
server thanks to ansible.

It checks the code set up an ssh key, install ansible and finally
disable the checking of the key by the host.
```ssh
> name: Deploy
>
> on:
>
>   workflow_run:
>
>     workflows: \[\"CI devops 2024\"\]
>
>     types:
>
>       - completed
>
> jobs:
>
>   deploy:
>
>     runs-on: ubuntu-latest
>
>     if: \${{ github.event.workflow_run.conclusion == \'success\' }}
>
>     steps:
>
>       - name: Checkout code
>
>         uses: actions/checkout@v2.5.0
>
>       - name: Setup SSH and known hosts
>
>         uses: webfactory/ssh-agent@v0.5.4
>
>         with:
>
>           ssh-private-key: \${{ secrets.SSH_PRIVATE_KEY }}
>
>       - name: Install Ansible
>
>         run: \|
>
>           sudo apt-get update
>
>           sudo apt-get install -y ansible
>
>       - name: Disable host key checking
>
>         run: echo \"ANSIBLE_HOST_KEY_CHECKING=False\" \>\>
> \$GITHUB_ENV
>
>       - name: Deploy to production server
>
>         run: \|
>
>           ansible-playbook -i inventories/setup.yml
> my-project/ansible/playbook.yml
```