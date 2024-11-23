### Dockerfile로 배포하기(Dockerfile)

    - myapp
    mvn clean package
    
    ### Dockerfile #######################
    FROM openjdk:17-slim
    ARG JAR_FILE=target/*.jar
    COPY ${JAR_FILE} app.jar
    ENTRYPOINT ["java","-jar","./app.jar"]
    ######################################
    
    docker build -t myapp:0.1 .
    docker run --name myapp -d -p 8080:8080 myapp:0.1

    - catalog-service
    # docker run으로 mysql container 생성하기
    docker network create -d bridge mynet
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog_dev --net mynet -d -p 3306:3306 mysql:latest

    => ./gradlew clean bootBuildImage (Buildpack을 사용: Permission denied경우, chmod 755 ./gradlew)
       mvn spring-boot:build-image(Maven 프로젝트인경우)
    
    # docker run으로 배포하기
    docker network create -d bridge mynet
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog --net mynet -d -p 3306:3306 mysql:latest
    docker run --name catalog-service-jpa -d -p 9001:9001 -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/polardb_catalog  --net mynet catalog-service-jpa:0.0.1-SNAPSHOT

### Docker Compose로 배포하기(docker-compose.yml)
    # docker compose으로 배포하기
    docker compose up -d
    docker compose down
