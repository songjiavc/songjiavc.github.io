### sonar 简介
SonarQube is an open source platform for continuous inspection of code quality.
### 环境搭建配置
    https://hub.docker.com/_/sonarqube/
    docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube
    登陆用户名密码：admin/admin
### 访问路径
http://192.168.0.208:9000/issues?resolved=false
