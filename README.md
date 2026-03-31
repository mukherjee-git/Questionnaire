# Java Maven CI/CD Demo (Jenkins → Sonar → Docker → Nexus/JFrog → Argo CD)

This is a **realistic sample project** you can push to Git and point a Jenkins Pipeline job at.

## What the pipeline does
1. Maven build  
2. Unit tests (Surefire)  
3. Functional tests (Failsafe, `*IT.java`)  
4. Performance test (k6 script hitting the running container)  
5. Sonar scan + Quality Gate  
6. Docker build  
7. Push Docker image to **Nexus** and **JFrog Artifactory**  
8. Update GitOps repo (image tag for both overlays: `nexus` & `jfrog`)  
9. Trigger Argo CD sync for both Applications  

## Local run
```bash
mvn clean test
mvn -Pfunctional-tests verify
mvn spring-boot:run
