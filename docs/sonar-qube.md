# Install sonar
```
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```
Sau đó, truy cập http://localhost:9000 (User/Pass mặc định là admin/admin)

# Running scan script
## Scan backend
```
mvn clean verify sonar:sonar ^
  -Dsonar.projectKey=vocup-be-appservice ^
  -Dsonar.host.url=http://localhost:9000 ^
  -Dsonar.login=sqp_fb06b788c06b180b40713ec2cecdfc9371cf83a1
```

## Scan FE
```
docker run --rm -e SONAR_HOST_URL="http://host.docker.internal:9000" -v "%cd%:/usr/src" sonarsource/sonar-scanner-cli -Dsonar.projectKey=vocup-fe-lms -Dsonar.token=sqp_c9fe4dcd7fa510c983b2f60235bb21f5b4713d9c -Dsonar.exclusions=node_modules/**,dist/**,build/**
```
