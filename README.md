# TP1 - PERON Guilhem 

## 1-1. Document your database

### Installation

1. **Créer un Réseau Docker** :

   ```sh
   docker network create app-network

2. **Construire l'Image Docker :** :
   ```sh
    docker build -t my-postgres-image .

3. **Démarrer le Conteneur PostgreSQL** :
   ```sh
   docker run -d \
    --name my-postgres-container \ #container name
    --network app-network \ #network
    -e POSTGRES_DB=db \ #database
    -e POSTGRES_USER=usr \ #user 
    -e POSTGRES_PASSWORD=pwd \ #password
    -v postgres_data:/var/lib/postgresql/data \ #volume
    my-postgres-image #ref image

4. **Démarrer Adminer** :
    ```sh
    docker run -d \ 
    --name adminer \ #container name
    --network app-network \ #network
    -p 8090:8080 \ #port
    adminer #image


### Fichiers necessaires pour créer l'image:

1. **Dockerfile** :

    ```sh
    #Utiliser l'image de base postgres:14.1-alpine
    FROM postgres:14.1-alpine

    #Définir les variables d'environnement pour PostgreSQL
    ENV POSTGRES_DB=db \
        POSTGRES_USER=usr \
        POSTGRES_PASSWORD=pwd

    #Copier les scripts SQL dans le répertoire d'initialisation de PostgreSQL
    COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
    COPY 02-InsertData.sql /docker-entrypoint-initdb.d/

2. **CreateScheme.sql** :
    ```sql
        CREATE TABLE public.departments
    (
    id      SERIAL      PRIMARY KEY,
    name    VARCHAR(20) NOT NULL
    );

    CREATE TABLE public.students
    (
    id              SERIAL      PRIMARY KEY,
    department_id   INT         NOT NULL REFERENCES departments (id),
    first_name      VARCHAR(20) NOT NULL,
    last_name       VARCHAR(20) NOT NULL
    );

3. **CreateScheme.sql** :
    ```sql
    INSERT INTO departments (name) VALUES ('IRC');
    INSERT INTO departments (name) VALUES ('ETI');
    INSERT INTO departments (name) VALUES ('CGP');

    INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
    INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
    INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
    INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');


### 1-2. Why do we need a multistage build? And explain each step of this dockerfile.

A multistage build in Docker allows you to use multiple `FROM` statements in a single Dockerfile, each representing a separate build stage. This is useful for optimizing Docker images by separating the build environment from the runtime environment. It helps reduce the size of the final image by discarding unnecessary build dependencies and artifacts.

    ```sh
    # Build
    FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY pom.xml .
    COPY src ./src
    RUN mvn package -DskipTests

    # Run
    FROM amazoncorretto:17
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

    ENTRYPOINT java -jar myapp.jar

### Explanation of each step in the Dockerfile:

1. **Build Stage (myapp-build):**
   - `FROM maven:3.8.6-amazoncorretto-17 AS myapp-build`: Specifies the base image for the build stage, which includes Maven and JDK 17.
   - `ENV MYAPP_HOME /opt/myapp`: Sets an environment variable `MYAPP_HOME` to the directory `/opt/myapp`.
   - `WORKDIR $MYAPP_HOME`: Sets the working directory inside the container to `/opt/myapp`.
   - `COPY pom.xml .`: Copies the `pom.xml` file from the local directory to the working directory in the container.
   - `COPY src ./src`: Copies the source code files from the local `src` directory to the `src` directory in the container.
   - `RUN mvn package -DskipTests`: Builds the Maven project, skipping tests, to generate the application JAR file.

2. **Deployment Stage:**
   - `FROM amazoncorretto:17`: Specifies the base image for the deployment stage, which includes JDK 17.
   - `ENV MYAPP_HOME /opt/myapp`: Sets an environment variable `MYAPP_HOME` to the directory `/opt/myapp`.
   - `WORKDIR $MYAPP_HOME`: Sets the working directory inside the container to `/opt/myapp`.
   - `COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar`: Copies the application JAR file generated in the build stage to the working directory in the deployment stage.
   - `ENTRYPOINT java -jar myapp.jar`: Specifies the default command to run when the container starts, which is to execute the application JAR file using Java.

This Dockerfile uses a multistage build to separate the build process (compilation and packaging of the application) from the deployment process (running the application). The build stage uses a Maven-based image to compile the source code and generate the application JAR file, while the deployment stage uses a lightweight JDK image to run the application. This helps create a smaller and more efficient Docker image by discarding unnecessary build dependencies and intermediate files.


