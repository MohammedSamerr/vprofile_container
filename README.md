# vprofile_container

> Dockerizing and composing a Java Spring legacy web app (vprofile) — multi-stage builds, dependency extraction, and compose-driven multi-container development.

## Project overview

This repository contains a Java Spring MVC web application (packaged with Maven) and supporting files showing how I practiced containerizing a multi-component app using Docker multi-stage builds and Docker Compose. The repo includes:

* Application source: `/src` (Spring MVC + JSP + Spring Security, JPA).
* Maven project file: `pom.xml`.
* Database dump: `/src/main/resources/db_backup.sql` (MySQL dump).
* Docker-related artifacts: `Docker-files/` folder and `compose.yaml`.
* CI/CD pipe skeleton: `Jenkinsfile` and `vagrant`, `ansible` experiment folders.

(Repo reference: the repository on GitHub.)

---

## Goals of this practice

1. Extract application build artifacts and dependencies cleanly, then produce a small runtime image.
2. Practice multi-stage Dockerfile patterns (build stage vs runtime stage) for a Java/Maven app.
3. Use Docker Compose to wire multiple services (app, MySQL) for local development and testing.
4. Demonstrate how to keep images small, reproducible, and suitable for CI/CD pipelines.

---

## Why multi-stage builds?

Multi-stage builds let you perform the heavy build steps (compiling, running Maven) in a builder image, then copy only the final artifact (the `.jar` or exploded webapp) into a slim runtime image. This reduces image size, surface area for vulnerabilities, and speeds up distribution.

---

## Recommended multi-stage Dockerfile (example)

Place this in `Docker-files/Dockerfile` (or at project root) — this is a generic, tested pattern for Maven + Spring Boot / Java apps that target JDK 11 (your repo lists JDK 11 in prerequisites):

```Dockerfile
# ---------- builder stage ----------
FROM maven:3.8.8-eclipse-temurin-11 AS builder
WORKDIR /build

# copy only pom first to cache dependencies
COPY pom.xml .
COPY src ./src

# build the project (skip tests when iterating locally)
RUN mvn -B -e -DskipTests package

# ---------- runtime stage ----------
FROM eclipse-temurin:11-jre-jammy
WORKDIR /app

# copy jar produced by the builder
COPY --from=builder /build/target/*.jar ./app.jar

# set JVM options with sane defaults; allow override via docker run -e JAVA_OPTS
ENV JAVA_OPTS="-Xms128m -Xmx512m"

EXPOSE 8080
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar /app/app.jar"]
```

Notes:

* If your project is a WAR deployed to Tomcat (not a fat jar), adapt the runtime image to Tomcat and copy the WAR into `webapps/`.
* The `mvn package` step produces the artifact in `target/` per `pom.xml` conventions.

---

## Extracting dependencies (if you prefer separating libs)

Sometimes you want to ship only the application classes and the `lib/` folder separately. A common technique:

* In the builder stage run `mvn dependency:copy-dependencies -DincludeScope=runtime -DoutputDirectory=libs` alongside `mvn package`.
* Copy `/build/target/app.jar` (or exploded classes) and `/build/libs/*` into the runtime image, then run `java -cp "libs/*:app.jar" your.main.Class`.

This is useful when you want to mount only config into the final image and still keep layers cache-friendly.

---

## Example docker-compose (compose.yaml)

A simple Compose setup to run the app + MySQL locally:

```yaml
version: '3.8'
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: accounts
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    volumes:
      - db_data:/var/lib/mysql
      - ./src/main/resources/db_backup.sql:/docker-entrypoint-initdb.d/db_backup.sql:ro
    ports:
      - "3306:3306"

  app:
    build:
      context: .
      dockerfile: Docker-files/Dockerfile
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/accounts?useSSL=false&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: apppass
    ports:
      - "8080:8080"
    restart: unless-stopped

volumes:
  db_data:
```

Notes:

* Compose mounts the SQL dump into `/docker-entrypoint-initdb.d/` so MySQL initializes the database on first run.
* Use `docker compose up --build` to build the app image from the multi-stage Dockerfile and run both services.

---

## Useful commands

* Build app image locally (from repo root):

  ```bash
  docker build -f Docker-files/Dockerfile -t vprofile-app:local .
  ```

* Run with compose (recommended):

  ```bash
  docker compose up --build
  ```

* Run app container only (detached):

  ```bash
  docker run -d -p 8080:8080 --name vprofile vprofile-app:local
  ```

* Inspect image layers (to confirm minimal runtime):

  ```bash
  docker history vprofile-app:local
  ```

* Clean up:

  ```bash
  docker compose down --volumes --rmi local
  ```

---

## CI/CD tips (Jenkinsfile present)

* Use the same multi-stage Dockerfile in your Jenkins pipeline: let Jenkins run `mvn -DskipTests package` in a Maven agent, build a runtime image, tag it with Git commit, and push to a registry.
* Avoid running Docker-in-Docker unless necessary. Use the Jenkins Docker plugin or Kaniko/buildpacks in pipeline for secure image builds.

---

## Suggested README improvements for the repo

1. Add a short project description and architecture diagram.
2. Move your `compose.yaml` to `docker-compose.yml` (standard name) or add doc that explains `compose.yaml` usage.
3. Include a sample `.env.example` describing DB credentials and ports.
4. Add explicit instructions for running the SQL dump and any migration steps.
5. Include a troubleshooting section (port conflicts, permissions on init SQL, Maven build fails—common issues).

---