# MicroShop Docker Setup Guide: From Zero to Hero

**Date**: June 22, 2025  
**Author**: Naresh Swami

## Introduction
This guide covers all steps to set up, build, push, and run the **MicroShop** project using Docker, from scratch. **MicroShop** is a microservices-based e-commerce application built with **Spring Boot 3.2.5**, **Spring Cloud 2023.0.1**, and **PostgreSQL**. It includes the following services running on a `microshop-network`:

- **Config Server** (Port 8888): Centralized configuration.
- **Eureka Server** (Port 8761): Service discovery.
- **User Service** (Port 8081): Manages user data.
- **Product Service** (Port 8082): Manages product data.
- **Gateway Service** (Port 8080): API gateway.
- **db** (Port 5432): PostgreSQL for User Service.
- **product-db** (Port 5433): PostgreSQL for Product Service.

The goal is to package services into Docker images, push them to **Docker Hub**, and run them on any PC (x86_64 or ARM64) using `docker-compose.yml`.

## Prerequisites

### Software Requirements
- **Docker**: Version `>= 20.10`.
- **Docker Compose**: Version `>= 2.0.0`.
- **Maven**: Version `>= 3.8`.
- **Git**: For cloning the project.
- **Docker Hub** account (e.g., `nareshswami`).

### Install Dependencies
On **Mac** (M1/M2):
```bash
brew install docker docker-compose git maven
```

On **Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install docker.io docker-compose-v2 git maven
```

On **Windows**:
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/).
- Install Maven via `choco` or manual download.

Verify installations:
```bash
docker --version
docker compose version
mvn -version
git --version
```

## Project Setup

### Clone or Setup Project
Clone the **MicroShop** repository (replace with your repo if different):
```bash
git clone https://github.com/nareshswami/microshop.git
cd microshop
```

Directory structure:
```
microshop/
├── config-server/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── eureka-server/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── user-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── product-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── gateway-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── docker-compose.yml
├── postgres-data/
└── product-postgres-data/
```

### Example Dockerfile
Each service (`config-server`, `eureka-server`, `user-service`, `product-service`, `gateway-service`) should have a `Dockerfile`:
```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Configuration Repository
Ensure **Config Server** points to your configuration repository (e.g., `https://github.com/nareshswami/microshop-config`). Example `user-service.yml`:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/userdb
    username: user
    password: password
  jpa:
    hibernate:
      ddl-auto: update
  application:
    name: user-service
server:
  port: 8081
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true
```
Update `product-service.yml` similarly:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://product-db:5432/productdb
    username: user
    password: password
  jpa:
    hibernate:
      ddl-auto: update
  application:
    name: product-service
server:
  port: 8082
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true
```

## Building Docker Images

### Maven Build
Compile and package all services:
```bash
cd /Users/nareshswami/Documents/microshop
mvn clean package -DskipTests -f config-server/pom.xml
mvn clean package -DskipTests -f eureka-server/pom.xml
mvn clean package -DskipTests -f user-service/pom.xml
mvn clean package -DskipTests -f product-service/pom.xml
mvn clean package -DskipTests -f gateway-service/pom.xml
```

### Docker Build (Single Architecture)
Build Docker images:
```bash
docker build -t nareshswami/microshop-config-server:latest ./config-server
docker build -t nareshswami/microshop-eureka-server:latest ./eureka-server
docker build -t nareshswami/microshop-user-service:latest ./user-service
docker build -t nareshswami/microshop-product-service:latest ./product-service
docker build -t nareshswami/microshop-gateway-service:latest ./gateway-service
```

### Docker Build (Multi-Architecture for M1/M2 Mac)
For compatibility with x86_64 and ARM64:
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t nareshswami/microshop-config-server:latest --push ./config-server
docker buildx build --platform linux/amd64,linux/arm64 -t nareshswami/microshop-eureka-server:latest --push ./eureka-server
docker buildx build --platform linux/amd64,linux/arm64 -t nareshswami/microshop-user-service:latest --push ./user-service
docker buildx build --platform linux/amd64,linux/arm64 -t nareshswami/microshop-product-service:latest --push ./product-service
docker buildx build --platform linux/amd64,linux/arm64 -t nareshswami/microshop-gateway-service:latest --push ./gateway-service
```

## Pushing Images to Docker Hub
Login and push images:
```bash
docker login
docker push nareshswami/microshop-config-server:latest
docker push nareshswami/microshop-eureka-server:latest
docker push nareshswami/microshop-user-service:latest
docker push nareshswami/microshop-product-service:latest
docker push nareshswami/microshop-gateway-service:latest
```

## Docker Compose Configuration
Save the following `docker-compose.yml` in `/Users/nareshswami/Documents/microshop`:
```yaml
services:
  db:
    container_name: microshop-postgres
    image: postgres:16
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=userdb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  product-db:
    container_name: microshop-product-postgres
    image: postgres:16
    volumes:
      - ./product-postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=productdb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  config-server:
    container_name: microshop-config-server
    image: nareshswami/microshop-config-server:latest
    ports:
      - "8888:8888"
    depends_on:
      db:
        condition: service_healthy
      product-db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:8888/actuator/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  eureka-server:
    container_name: microshop-eureka-server
    image: nareshswami/microshop-eureka-server:latest
    ports:
      - "8761:8761"
    depends_on:
      config-server:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:8761/actuator/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  user-service:
    container_name: microshop-user-service
    image: nareshswami/microshop-user-service:latest
    ports:
      - "8081:8081"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/userdb
      - SPRING_DATASOURCE_USERNAME=user
      - SPRING_DATASOURCE_PASSWORD=password
    depends_on:
      db:
        condition: service_healthy
      config-server:
        condition: service_healthy
      eureka-server:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:8081/actuator/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  product-service:
    container_name: microshop-product-service
    image: nareshswami/microshop-product-service:latest
    ports:
      - "8082:8082"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://product-db:5432/productdb
      - SPRING_DATASOURCE_USERNAME=user
      - SPRING_DATASOURCE_PASSWORD=password
    depends_on:
      product-db:
        condition: service_healthy
      config-server:
        condition: service_healthy
      eureka-server:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:8082/actuator/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

  gateway-service:
    container_name: microshop-gateway-service
    image: nareshswami/microshop-gateway-service:latest
    ports:
      - "8080:8080"
    depends_on:
      config-server:
        condition: service_healthy
      eureka-server:
        condition: service_healthy
      user-service:
        condition: service_healthy
      product-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --spider http://localhost:8080/actuator/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: on-failure
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - microshop-network

networks:
  microshop-network:
    driver: bridge
```

## Running Locally
Clear old volumes (if any):
```bash
docker volume rm microshop_postgres-data microshop_product-postgres-data
```

Run Docker Compose:
```bash
cd /Users/nareshswami/Documents/microshop
docker compose down
docker compose up --build -d
```

## Running on Another PC

### Install Docker
On **Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install docker.io docker-compose-v2
```

On **Windows/Mac**:
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/).

### Transfer docker-compose.yml
Copy `docker-compose.yml` to the new PC:
```bash
scp /Users/nareshswami/Documents/microshop/docker-compose.yml user@other-pc:/path/to/microshop/
```

### Pull Images
```bash
docker login
docker pull nareshswami/microshop-config-server:latest
docker pull nareshswami/microshop-eureka-server:latest
docker pull nareshswami/microshop-user-service:latest
docker pull nareshswami/microshop-product-service:latest
docker pull nareshswami/microshop-gateway-service:latest
```

### Run Docker Compose
```bash
cd /path/to/microshop
docker compose up -d
```

## Testing
Verify services:
```bash
docker compose ps
curl http://localhost:8888/actuator/health
curl http://localhost:8761
curl http://localhost:8081/users/1
curl http://localhost:8080/users/1
```

Check logs:
```bash
docker compose logs config-server
docker compose logs eureka-server
docker compose logs user-service
docker compose logs product-service
docker compose logs gateway-service
docker compose logs db
docker compose logs product-db
```

Test PostgreSQL:
```bash
docker exec -it microshop-postgres psql -U user -d userdb -c "SELECT 1;"
docker exec -it microshop-product-postgres psql -U user -d productdb -c "SELECT 1;"
```

## Troubleshooting

### Resources Not Allowed Error
If `services.gateway-service Additional property resources is not allowed` occurs:
- Replaced `resources` with `mem_limit`, `mem_reservation`, and `cpus` in `docker-compose.yml`.
- Example:
  ```yaml
  mem_limit: 1024m
  mem_reservation: 512m
  cpus: "1.0"
  ```

### PostgreSQL Role Issues
If `role "root" does not exist` occurs:
```bash
docker volume rm microshop_postgres-data microshop_product-postgres-data
docker compose up -d
```

### Manual Service Restart
If services require manual restart:
```bash
docker compose restart user-service product-service gateway-service
docker compose logs user-service
```

### Docker Version
```bash
docker compose version
docker --version
```

### Resources
```bash
docker info --format '{{.CPUs}}'
docker info --format '{{.MemTotal}}'
```

### Network Issues
```bash
docker network inspect microshop-network
docker exec -it microshop-gateway-service ping user-service
```

### Configuration Issues
Refresh **Config Server**:
```bash
curl -X POST http://localhost:8888/actuator/refresh
```

### ServletException or 404 Errors
- Ensure `spring-cloud-starter-gateway` in `gateway-service/pom.xml` and remove `spring-boot-starter-web`.
- Verify **User Service** endpoint (`GET /users/{id}`) and **Gateway** routing (`lb://user-service`).

## Conclusion
This guide covers all Docker commands and steps to set up **MicroShop** from scratch, addressing issues like `resources not allowed`, `role "root"`, manual restarts, and `404/ServletException`. For further details, check the project's GitHub repository or contact the developer.
