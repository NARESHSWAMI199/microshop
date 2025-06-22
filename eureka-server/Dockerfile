FROM openjdk:17-jdk-slim
RUN apt-get update && apt-get install -y iputils-ping
COPY target/EurekaServer.jar EurekaServer.jar
ENTRYPOINT ["java", "-jar", "EurekaServer.jar"]