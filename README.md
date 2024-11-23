### Dockerfile로 배포하기
    mvn clean package
    
    ### Dockerfile #######################
    FROM openjdk:17-slim
    ARG JAR_FILE=target/*.jar
    COPY ${JAR_FILE} app.jar
    ENTRYPOINT ["java","-jar","./app.jar"]
    ######################################
    
    docker build -t myapp:0.1 .
    docker run --name myapp -d -p 8080:8080 myapp:0.1
