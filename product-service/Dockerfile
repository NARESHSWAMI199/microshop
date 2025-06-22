FROM openjdk:17-jdk-slim
RUN apt-get update && apt-get install -y iputils-ping
COPY target/MicroShopProduct.jar MicroShopProduct.jar
ENTRYPOINT ["java" , "-jar" , "MicroShopProduct.jar"]