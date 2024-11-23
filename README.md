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
    * 접속 : http://localhost:8080/hello

    - catalog-service & database(mysql)
    # docker run으로 mysql container 생성하기
    docker network create -d bridge mynet
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog_dev --net mynet -d -p 3306:3306 mysql:latest

    => ./gradlew clean bootBuildImage (Buildpack을 사용: Permission denied경우, chmod 755 ./gradlew)
       mvn spring-boot:build-image(Maven 프로젝트인경우)
    
    # docker run으로 배포하기
    docker network create -d bridge mynet(이전에 만든 네트워크가 있으면 실행할 필요가 없음)
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog --net mynet -d -p 3306:3306 mysql:latest
    docker run --name catalog-service -d -p 9001:9001 -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/polardb_catalog  --net mynet catalog-service:0.0.1-SNAPSHOT
    * 접속 : http://localhost:9001/books

### Docker Compose로 배포하기(docker-compose.yml)
    - catalog-service & database(mysql)
    ### docker-compose.yml #########################
    services:
    
      # Applications
      catalog-service:
        depends_on:
          - polar-mysql
        image: "catalog-service:0.0.1-SNAPSHOT"
        container_name: "catalog-service"
        restart: always
        ports:
          - 9001:9001
        environment:
          - SPRING_DATASOURCE_URL=jdbc:mysql://polar-mysql:3306/polardb_catalog
          - SPRING_DATASOURCE_USERNAME=user1
          - SPRING_DATASOURCE_PASSWORD=1234
      
      # Backing Services
      polar-mysql:
        image: "mysql:latest"
        container_name: "polar-mysql"
        ports:
          - 3306:3306
        environment:
          - MYSQL_ROOT_PASSWORD=1234
          - MYSQL_USER=user1
          - MYSQL_PASSWORD=1234
          - MYSQL_DATABASE=polardb_catalog
    ################################################
    docker compose up -d
    docker compose down

### Kubernetes에 배포하기(deployment.yml, service.yml)

    ### deployment.yml #############################
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: catalog-service
      labels:
        app: catalog-service
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: catalog-service
      template:
        metadata:
          labels:
            app: catalog-service
        spec:
          containers:
            - name: catalog-service
              image: catalog-service
              imagePullPolicy: IfNotPresent
              lifecycle:
                preStop:
                  exec:
                    command: [ "sh", "-c", "sleep 5" ]
              ports:
                - containerPort: 9001
              env:
                - name: BPL_JVM_THREAD_COUNT
                  value: "50"
                - name: SPRING_DATASOURCE_URL
                  value: jdbc:mysql://polar-postgres/polardb_catalog
                - name: SPRING_PROFILES_ACTIVE
                  value: testdata
    ################################################

    ### service.yml ################################
    apiVersion: v1
    kind: Service
    metadata:
      name: catalog-service
      labels:
        app: catalog-service
    spec:
      type: ClusterIP
      selector:
        app: catalog-service
      ports:
      - protocol: TCP
        port: 80
        targetPort: 9001  
    ################################################

    kubectl apply -f deployment.yml
    kubectl apply -f service.yml
