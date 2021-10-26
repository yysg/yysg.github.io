---
title: "SonarCube 使用指南"
date: 2021-10-25T15:02:24+08:00
lastmod: 2021-10-25T15:02:24+08:00
draft: false
tags: ["devops", "test", "质量管理"]
categories: ["devops"]
author: "yang18"
---

### SonarCube 使用指南

![sonar_overview](https://cdn.jsdelivr.net/gh/yysg/blog-img/root/sonar_overview.png)

1. 安装 docker 版本的 SonarCube。创建`docker-compose.yml` 文件

   ```yml
   version: "3"

   services:
     sonarqube:
       image: sonarqube:8-community
       depends_on:
         - db
       environment:
         SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
         SONAR_JDBC_USERNAME: sonar
         SONAR_JDBC_PASSWORD: sonar
       volumes:
         - sonarqube_data:/opt/sonarqube/data
         - sonarqube_extensions:/opt/sonarqube/extensions
         - sonarqube_logs:/opt/sonarqube/logs
         - sonarqube_temp:/opt/sonarqube/temp
       ports:
         - "9000:9000"
     db:
       image: postgres:12
       environment:
         POSTGRES_USER: sonar
         POSTGRES_PASSWORD: sonar
       volumes:
         - postgresql:/var/lib/postgresql
         - postgresql_data:/var/lib/postgresql/data

   volumes:
     sonarqube_data:
     sonarqube_extensions:
     sonarqube_logs:
     sonarqube_temp:
     postgresql:
     postgresql_data:
   ```

2. 启动

   > <font size=4 color=red>如果启动失败，看这里</font>
   >
   > 1. sonarcube production 级别的配置需要设置一些参数
   >
   >    Because SonarQube uses an embedded Elasticsearch, make sure that your Docker host configuration complies with the [Elasticsearch production mode requirements](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode) and [File Descriptors configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html).
   >
   > 2. 本地（macos）docker 的配置提高些

   1. `docker-compose up -d `
   2. 访问 `localhost:9000` 用户名/密码 admin

3. 后台(使用 mvn 执行，需要先安装 maven)

   ```
   mvn sonar:sonar \
     -Dsonar.projectKey=backend \
     -Dsonar.host.url=http://localhost:9000 \
     -Dsonar.login=0f9efa0b98051368fce8e2974ea0332b2b168fda
   ```

4. 前台(使用 dockersonar-scanner-cli 执行)

   ```
   docker run \
       --rm \
       -e SONAR_HOST_URL="http://192.168.43.172:9000" \
       -v "/Users/yangjun/develop/poc/demo_webfront:/usr/src" \
       sonarsource/sonar-scanner-cli
   ```

### Jenkins 使用 SonarQube 插件（free style project）

1. 必要插件： node， SonarQube Scanner

2. manage jenkins -> config system -> SonarQube servers

   <img src="https://cdn.jsdelivr.net/gh/yysg/blog-img/root/sonar_jenkins_sonar_server.png" alt="sonar_jenkins_sonar_server" style="zoom:50%;" />

3. 新建项目

   ![sonar_jenkind_env_build](https://cdn.jsdelivr.net/gh/yysg/blog-img/root/sonar_jenkind_env_build.png)

   ![sonar_jenkins_build_option](https://cdn.jsdelivr.net/gh/yysg/blog-img/root/sonar_jenkins_build_option.png)

   ![sonar_jebkins_build](https://cdn.jsdelivr.net/gh/yysg/blog-img/root/sonar_jebkins_build.png)

⚠️ Execute SonarQube Scanner 构建时 Analysis properties 是主要内容

​ 主要内容应该是 SonarQube 构建项目时提供的参数.

```
sonar.projectKey=backend
sonar.java.binaries=$WORKSPACE // 这个是java项目需要的
```

### Jenkins 使用 SonarQube 插件（pipline）

1. front piplin script

   ```
   pipeline {
       agent any
       tools {nodejs "node12"}
       stages {
           stage('SCM') {
               steps{
                   git([url: 'git@github.com:leonsite/demo_webfront.git', branch: 'master', credentialsId: '43dd0fe7-8410-4934-bf65-571e149f31f7'])
               }
           }
           stage('SonarQube Analysis') {
               steps{
                   script{
                       def scannerHome = tool 'SonarQubeScanner';
                       withSonarQubeEnv('SonarQube') {
                         // http://192.168.43.172:9000/ is url of sonar server
                           sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=front -Dsonar.projectName=front -Dsonar.sourceEncoding=UTF-8 -Dsonar.host.url=http://192.168.43.172:9000/ "
                       }
                   }

               }
           }
           stage('SonarQube Quality Gate') {
               steps{
                   script{
                        timeout(time: 1, unit: 'HOURS') {
                           def qg = waitForQualityGate()
                           if (qg.status != 'OK') {
                             error "Pipeline aborted due to quality gate failure: ${qg.status}"
                           } else {
                             echo qg.status
                           }
                       }
                   }
               }
           }
      }
   }

   ```

   ⚠️

   如果想要使用 SonarQube Quality Gate 的返回结果，需要在 sonarqube 页面配置 webhook

   Administration -> Configuration -> Webhooks -> create `http://${jenkins url}/sonarqube-webhook/`

2. backend pipline script

   ```
   pipeline {
       agent any
       tools {
           maven "M3"
       }
       stages {
           stage('SCM') {
               steps{
                   git([url: 'git@github.com:leonsite/demo_backend.git', branch: 'master', credentialsId: '43dd0fe7-8410-4934-bf65-571e149f31f7'])

               }
           }
           stage('SonarQube Analysis') {
               steps{
                   withSonarQubeEnv('SonarQube') {
                       sh "mvn clean install -Dmaven.test.skip=true sonar:sonar -Dsonar.projectKey=backend -Dsonar.projectName=backend -Dsonar.sourceEncoding=UTF-8 -Dsonar.exclusions=src/test/** -Dsonar.host.url=http://192.168.43.172:9000/ "
                   }
               }
           }
    }
   }

   ```
