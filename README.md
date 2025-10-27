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

# Jenkins + Tomcat + Maven — Debug & Quick-Reference Guide

**Author:** Priyanka Hotkar  
**Purpose:** Concise step-by-step checklist of the main setup steps you followed and the primary debugging points to resolve common issues when using Jenkins (in Docker) to build a Java app and deploy to Tomcat (in Docker or local).

> This file summarizes the minimal commands, checks, and fixes that resolved the issues encountered during the pipeline setup and deployment workflow.

---

## Quick summary of the workflow

1. Host: Windows (PowerShell).
2. Jenkins runs in Docker container on Docker network `jenkins`.
3. Tomcat runs in Docker container on same Docker network; mapped host port `8888` (container 8080 -> host 8888).
4. GitHub repo contains Maven webapp (packaging `war`) and `Jenkinsfile` (using Tomcat Manager deploy).
5. Jenkins builds WAR (`target/*.war`) and deploys to Tomcat via manager API (`curl` or `tomcat-maven-plugin`) or by copying WAR to shared volume.

---

## Files in repository (reference)

```
src/main/java/com/example/HelloServlet.java
src/main/webapp/index.jsp
src/main/webapp/WEB-INF/web.xml
pom.xml          # packaging = war
Jenkinsfile      # CI/CD pipeline (build + deploy)
README.md
```

---

## Essential commands (copy-paste)

### Docker: run Tomcat on port 8888 and join `jenkins` network

```bash
docker run -d --name tomcat --network jenkins -p 8888:8080 tomcat:9.0.82-jdk17-temurin
```

If you used a different tag earlier and webapps were empty, use the full image tag above.

### Docker: run Jenkins on `jenkins` network and mount shared volume (optional)

```bash
docker volume create tomcat_webapps
docker run -d --name jenkins --network jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home -v tomcat_webapps:/opt/tomcat/webapps jenkins/jenkins:lts
```

This mounts the same volume `tomcat_webapps` into Jenkins at `/opt/tomcat/webapps`, so Jenkins can `cp` WAR into that path.

### If you prefer manager deployment (no volume)

Edit `tomcat-users.xml` to add a manager user:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```

Then restart Tomcat:

```bash
docker restart tomcat
```

### Test Tomcat Manager access (host)

Open browser: `http://localhost:8888/manager/html` → login `admin` / `admin123`

### Copy WAR from Windows host into Tomcat container

(Use container id/name from `docker ps`)

```powershell
docker cp "C:\path\to\project\target\myapp.war" <container_id>:/usr/local/tomcat/webapps/
docker exec -it <container_id> /usr/local/tomcat/bin/shutdown.sh
docker exec -it <container_id> /usr/local/tomcat/bin/startup.sh
```

### Deploy via Manager API from Jenkins (example `curl`)

Inside Jenkins (container), deploy to Tomcat container reachable by name (`tomcat:8080`):

```bash
curl -T target/myapp.war "http://admin:admin123@tomcat:8080/manager/text/deploy?path=/myapp&update=true"
```

**Important:** Jenkins must be able to resolve `tomcat` (same Docker network).

---

## Jenkinsfile — recommended minimal (manager deploy)

Use this in your repo's `Jenkinsfile` (Declarative):

```groovy
pipeline {
  agent any
  tools { maven 'Maven3' jdk 'Java19' }

  stages {
    stage('Checkout') { steps { git branch: 'main', url: 'https://github.com/your/repo.git' } }
    stage('Build')    { steps { sh 'mvn clean package -DskipTests' } }
    stage('Deploy')   {
      steps {
        sh 'curl -T target/*.war "http://admin:admin123@tomcat:8080/manager/text/deploy?path=/myapp&update=true"'
      }
    }
  }
}
```

---

## Common problems & how to debug (check these in order)

### 1) Jenkins fails to fetch repo / branch issues

- Symptom: `ERROR: Couldn't find any revision to build`
- Fix: Ensure job's Branch Specifier is `*/main` (or correct branch). Ensure Jenkinsfile path correct.

### 2) Maven / JDK tool names not configured

- Symptom: `Tool type "maven" does not have an install of "Maven3" configured`
- Fix: In Jenkins → Manage Jenkins → Global Tool Configuration → add Maven and JDK installations with the exact names used in the Jenkinsfile (e.g., `Maven3`, `Java19`).

### 3) Jenkins inside container can't reach Tomcat host by `localhost`

- Symptom: curl to `http://localhost:8888` from Jenkins fails; but host browser works.
- Fix: Use container hostname `tomcat` and internal port `8080` in Jenkinsfile (e.g., `http://tomcat:8080/...`) since both containers are on same Docker network.

### 4) `manager` app missing / webapps empty

- Symptom: `ls /usr/local/tomcat/webapps` shows empty folder (no manager, ROOT etc.)
- Fix: Use a Tomcat image/tag that includes default webapps (e.g., `tomcat:9.0.82-jdk17-temurin`) or copy `webapps.dist/*` into `webapps` and restart.

### 5) 404 after deployment

- Symptom: Jenkins logs show successful `curl` but browser returns 404.
- Checklist:
  - Confirm WAR exists inside container: `docker exec -it tomcat ls -l /usr/local/tomcat/webapps`
  - Confirm Tomcat exploded WAR into folder: `ls /usr/local/tomcat/webapps/myapp`
  - Check Tomcat logs: `docker exec -it tomcat tail -n 50 /usr/local/tomcat/logs/catalina.out`
  - If WAR not present, either Jenkins uploaded to wrong path or deployment failed silently.
  - If WAR present but not exploded: check permissions and catalina logs for errors.

### 6) Batch vs Shell commands

- Symptom: `Batch scripts can only be run on Windows nodes`
- Fix: Use `sh` steps for Linux-based Jenkins agents (default inside Docker). Use `bat` only on Windows agents.

---

## Useful commands (one-liners)

Find Jenkins container name:

```bash
docker ps
```

Check mounts for Tomcat:

```powershell
docker inspect tomcat --format="{{json .Mounts}}"
```

Copy file from Jenkins workspace to host:

```powershell
docker cp jenkins:/var/jenkins_home/workspace/YourJob/target/myapp.war "C:\Users\PRIYANKA\myapp.war"
```

Check Tomcat logs:

```bash
docker exec -it tomcat tail -n 200 /usr/local/tomcat/logs/catalina.out
```

Restart Tomcat:

```bash
docker exec -it tomcat /usr/local/tomcat/bin/shutdown.sh
docker exec -it tomcat /usr/local/tomcat/bin/startup.sh
```

---

## Final checklist (before submitting assignment)

- [ ] Repo contains `Jenkinsfile`, `pom.xml`, webapp sources
- [ ] Jenkins job configured for repo/branch and tools configured
- [ ] Tomcat container running on same Docker network
- [ ] `tomcat-users.xml` contains manager user and password
- [ ] Jenkins pipeline builds WAR and either copies WAR to shared volume or uses manager API
- [ ] Access app at `http://localhost:8888/<context>` and verify response

---
