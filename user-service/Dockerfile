FROM openjdk:17-jdk-slim
RUN apt-get update && apt-get install -y iputils-ping
COPY target/user-service.jar user-service.jar
ENTRYPOINT ["java", "-jar", "/user-service.jar"]