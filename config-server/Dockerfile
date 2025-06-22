FROM openjdk:17-jdk-slim
RUN apt-get update && apt-get install -y iputils-ping
COPY target/config-server.jar config-server.jar
ENTRYPOINT ["java", "-jar", "config-server.jar"]