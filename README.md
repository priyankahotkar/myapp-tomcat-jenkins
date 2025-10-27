# MyApp - Jenkins + Maven + Tomcat CI/CD Demo

This repository contains a simple Java web application (WAR) that demonstrates automated build
and deployment using Jenkins and Tomcat.

## Project structure
```
myapp/
├── src/main/java/com/example/HelloServlet.java
├── src/main/webapp/index.jsp
├── src/main/webapp/WEB-INF/web.xml
├── pom.xml
└── Jenkinsfile
```

## How to build locally
Requirements: Java 11+, Maven
```
mvn clean package
# WAR will be created at target/myapp.war
```

## Run Tomcat using Docker (recommended for testing)
```
docker run -d --name tomcat-server -p 8080:8080 -v /opt/tomcat/webapps:/usr/local/tomcat/webapps tomcat:9.0
# Ensure /opt/tomcat/webapps exists on host and is writable by Jenkins (or use sudo during deploy)
```

## Jenkins setup
1. Install Jenkins (can run in Docker)
2. Ensure Jenkins has Maven and JDK configured (Maven3, Java19 names in Global Tool Configuration)
3. Mount Docker or provide access to Tomcat deploy directory:
   - If Tomcat is a container mapping host directory `/opt/tomcat/webapps`, ensure Jenkins user can write there:
```
sudo mkdir -p /opt/tomcat/webapps
sudo chown -R jenkins:jenkins /opt/tomcat/webapps
```
4. Create a Pipeline job pointing to this repository and branch `main`.
5. Run the pipeline — it will build `target/myapp.war` and copy it to Tomcat webapps directory.

## Access application
Open: http://<host>:8080/myapp/
Servlet path: http://<host>:8080/myapp/hello

---
Author: Priyanka Hotkar
