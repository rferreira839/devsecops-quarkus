# ===== Stage 1: build =====
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY app/pom.xml .
COPY app/src ./src
RUN mvn -q -e -DskipTests=false package

# ===== Stage 2: runtime =====
FROM eclipse-temurin:17-jre
ENV QUARKUS_PROFILE=prod \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
WORKDIR /app
COPY --from=build /workspace/target/quarkus-app/ ./
EXPOSE 8080
ENTRYPOINT ["java","-jar","quarkus-run.jar"]
