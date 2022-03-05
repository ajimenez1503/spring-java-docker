# spring-java-docker

Getting started of Spring Java with Docker.
Following the guide of https://docs.docker.com/language/java/

### Test the spring app

- Clone repo
```shell
git clone https://github.com/spring-projects/spring-petclinic.git
```
- Test the application without Docker
```shell
./mvnw spring-boot:run
```
 - Open http://localhost:8080
### Build your Java image
- Create a `Dockerfile` file
```shell
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline

COPY src ./src

CMD ["./mvnw", "spring-boot:run"]
```
- Create a `.dockerignore` file
```shell
target
```
- Build an image
```shell
DOCKER_BUILDKIT=1 docker build --tag java-docker .
```
- View local images
```shell
docker images
```
- Tag images
```shell
docker tag java-docker:latest java-docker:v1.0.0
docker images
docker rmi java-docker:v1.0.0
docker images
```

### Run your image as a container
```shell
docker run --rm -d -p 8080:8080 --name springboot-server java-docker
docker ps
```
- Check the server is up
```shell
curl --request GET \
--url http://localhost:8080/actuator/health \
--header 'content-type: application/json'
```

### Run a database in a container
- Create the volume
```shell
docker volume create mysql_data
docker volume create mysql_config
```
- Create the network
```shell
docker network create mysqlnet
```
- Run MySql from a container
```shell
docker run -it --rm -d -v mysql_data:/var/lib/mysql \
-v mysql_config:/etc/mysql/conf.d \
--network mysqlnet \
--name mysqlserver \
-e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic \
-e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=petclinic \
-p 3306:3306 mysql:8.0.23
```
- Build and run the Java microservice docker
```shell
docker build --tag java-docker .

docker run --rm -d \
--name springboot-server \
--network mysqlnet \
-e MYSQL_URL=jdbc:mysql://mysqlserver/petclinic \
-p 8080:8080 java-docker

curl  --request GET \
  --url http://localhost:8080/vets \
  --header 'content-type: application/json'
```

### Use Compose to develop locally
- Create a new file `docker-compose.dev.yml`
```shell
version: '3.8'
services:
  petclinic:
    build:
      context: .
    ports:
      - 8000:8000
      - 8080:8080
    environment:
      - SERVER_PORT=8080
      - MYSQL_URL=jdbc:mysql://mysqlserver/petclinic
    volumes:
      - ./:/app
    command: ./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000"

  mysqlserver:
    image: mysql:8.0.23
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - mysql_data:/var/lib/mysql
      - mysql_config:/etc/mysql/conf.d
volumes:
  mysql_data:
  mysql_config:
```
- Run the containers:
```shell
docker-compose -f docker-compose.dev.yml up --build

curl  --request GET \
  --url http://localhost:8080/vets \
  --header 'content-type: application/json'
```

### Run your tests
```shell
docker run -it --rm --name springboot-test java-docker ./mvnw test
```
- Udpate dockerFile
```
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13 as base

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline
COPY src ./src

FROM base as test
RUN ["./mvnw", "test"]

FROM base as development
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql", "-Dspring-boot.run.jvmArguments='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000'"]

FROM base as build
RUN ./mvnw package

FROM openjdk:11-jre-slim as production
EXPOSE 8080

COPY --from=build /app/target/spring-petclinic-*.jar /spring-petclinic.jar

CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/spring-petclinic.jar"]
```
- Run the test as part of the build
```shell
docker build -t java-docker --target test .
```