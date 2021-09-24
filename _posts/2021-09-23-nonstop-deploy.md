---
layout: single
title: "[Dev] CI/CD 적용하기 - 3. 무중단 배포"
categories: [CodeDeploy]
tags: [Dev, CodeDeploy, AWS, Spring Boot]

date: 2021-09-03
last_modified_at: 2021-09-06
---

안녕하세요. 이번 게시글에서는 _CI/CD 적용하기_ 시리즈 세 번째 파트, 무중단 배포에 대해서 알아보겠습니다.

_CI/CD 적용하기_ 시리즈 두 번째 파트, [CI/CD 적용하기 - 2. 배포 자동화](https://leo0842.github.io/codedeploy/code-deploy/) 에 이어서 작성합니다.

> 개발 환경
> - Spring Boot 2.4.5
> - Gradle 6.8.3
> - Travis CI
> - AWS EC2, IAM, S3, Role, CodeDeploy
> - Nginx
> - Docker 3.9

현재 배포 서버 구동은 docker compose 로 실행되고 있습니다.

여기에 nginx 의 로드 밸런싱 기능과 블루/그린 방식을 이용하여 무중단 배포를 적용하겠습니다.

전체적인 무중단 배포의 흐름은 다음과 같습니다.

1. ec2 배포 서버에 jar 파일을 전달하고 deploy.sh 파일 실행 ([CI/CD 적용하기 - 2. 배포 자동화](https://leo0842.github.io/codedeploy/code-deploy/) 에서 했던 과정)
2. 현재 구동되는 서버의 플래그를 확인(블루/그린)하고 구동되지 않던 서버 구동(일시적으로 두 개의 서버 구동)
3. 새로 구동한 서버가 뜨면 기존에 구동되던 서버를 내림

이 과정에서 Nginx 의 로드밸런스 기능을 통해 두 개의 서버를 다 바라보고 있고 하나의 서버가 죽으면 나머지 하나만 바라보기때문에 중단 없이 서버가 운영될 수 있습니다.

### 플래그 설정

먼저 앱에서 환경 변수로 application.yml 파일에 플래그를 설정하겠습니다.

```yaml
deploy:
  name: ${DEPLOY_NAME}
```

deploy.name 에 blue 또는 green 이 두 개의 앱 각각에 설정되어 추후 어떤 서버가 구동되는지 확인하는 플래그입니다.

앞서 설정한 환경변수를 확인할 수 있는 로직을 간단히 만들겠습니다.

```java
@RestController
@RequiredArgsConstructor
public class FlagController {

  private final Environment env;

  @GetMapping("/health")
  public String checkHealth() {

    return env.getProperty("deploy.name");
  }
}
```

![인텔리제이 환경변수](/assets/images/intellij-env.png)

위와 같이 인텔리제이에 플래그 환경 변수를 설정하고 로컬에서 설정한 플래그가 잘 나오는지 테스트를 해보겠습니다.


인텔리제이에 해당 환경 변수에 값을 넣고 테스트를 해보겠습니다.

![curl 로 확인](/assets/images/curl-test.png)

curl 로 확인 결과 설정했던 환경 변수가 잘 나옵니다.

이제 ec2 배포 서버 환경으로 자리를 옮겨 나머지 작업을 하겠습니다.

### EC2

지난 포스트에서 작성했던 deploy.sh 은 그저 docker compose 파일을 내리고 띄우는 작동만 하였습니다.

이번 포스트에서는 해당 deploy.sh 이 조금 더 다양한 작업을 수행할 수 있도록 바꿉니다.

#### deploy.sh

```shell
#!/bin/bash

REPOSITORY=/home/ec2-user/app/travis/build/ # --- 1
DOCKER_REPOSITORY=/home/ec2-user/study-pot/nginx/

echo "JAR_FILE REPOSITORY = $REPOSITORY"
echo "---"
echo "DOCKER_REPOSITORY = $DOCKER_REPOSITORY"

JAR_FILE=$(ls $REPOSITORY |grep 'study-pot-api.jar') # --- 2

echo "docker compose 파일이 있는 곳으로 이동합니다."
cd ~/study-pot/nginx/ 
APP_BLUE=$(sudo docker-compose -p app-blue -f docker-compose.blue.yml ps | grep Up); # --- 3

FIND=""; # --- 4
CHECK_URL="도메인명/health";

if [ -z "$APP_BLUE" ]; then # --- 5.1
  echo "blue 서버를 띄웁니다.";
  echo "sudo docker-compose -p app-blue -f docker-compose.blue.yml up -d";
  sudo docker-compose -p app-blue -f docker-compose.blue.yml up -d # --- 6
  sleep 30 # --- 7.1
  for count in {1..5}
  do
    CHECK_BLUE=$(curl -s CHECK_URL | grep blue); # --- 8.1
    if [ -z "$CHECK_BLUE" ]; then # --- 8.2
      echo "loading... $count"
    else
      echo "서버가 성공적으로 구동되었습니다."
      FIND="FIND" # --- 8.3
      break
    fi
    sleep 5 # --- 7.2
  done
  if [ -z "$FIND" ]; then # --- 9.1
    echo "앱 실행에 실패하였습니다. green 서버를 계속 구동합니다."
    sudo docker-compose -p app-blue -f docker-compose.blue.yml down # --- 9.2
  else
    echo "앱 실행에 성공하였습니다. green 서버를 내립니다."
    echo "sudo docker-compose -p app-green -f docker-compose.green.yml down"
    sudo docker-compose -p app-green -f docker-compose.green.yml down # --- 9.3
  fi
else # --- 5.2
  echo "green 서버를 띄웁니다.";
  echo "sudo docker-compose -p app-green -f docker-compose.green.yml up -d";
  sudo docker-compose -p app-green -f docker-compose.green.yml up -d
  sleep 30
  for count in {1..5}
  do
    CHECK_GREEN=$(curl -s CHECK_URL | grep green);
    if [ -z "$CHECK_GREEN" ]; then
      echo "loading... $count"
    else
      echo "서버가 성공적으로 구동되었습니다."
      FIND="FIND"
      break
    fi
    sleep 5
  done
  if [ -z "$FIND" ]; then
    echo "앱 실행에 실패하였습니다. blue 서버를 계속 구동합니다."
    sudo docker-compose -p app-green -f docker-compose.green.yml down
  else
    echo "앱 실행에 성공하였습니다. blue 서버를 내립니다."
    echo "sudo docker-compose -p app-blue -f docker-compose.blue.yml down"
    sudo docker-compose -p app-blue -f docker-compose.blue.yml down
  fi
fi
```
1. REPOSITORY 와 DOCKER_REPOSITORY 를 변수로 등록합니다. /home/ec2-user/ 와 같이 절대 경로로 설정합니다.
    

2. REPOSITORY 폴더의 경로에서 CodeDeploy 로부터 받은 jar 파일을 변수로 지정합니다.

    
3. docker-compose 의 ls 명령어를 통해 blue 가 떠있는지 확인합니다.
   

4. FIND 변수는 앱 서버 구동이 성공했는지 실패했는지 판단할 플래그 변수입니다.


5. 3의 APP_BLUE 에 아무 것도 저장되지 않았다면 blue 서버를 띄워야 합니다. 저장되었다면 5.2 의 green 서버를 띄웁니다.


6. -p 옵션은 프로젝트 명(컨테이너 이름과 함께 쓰입니다)을 지정하는 옵션이고 -f 옵션은 docker compose 파일 명을 지정하는 옵션입니다. 


7. 30초 동안 재우고 for 구문으로 5번 반복합니다. 반복 시 7.2 라인의 5초씩 더 재우면 최대 55초의 시간을 서버 구동하는 데에 가집니다. 


8.
  - /health 경로를 확인하여 앱에서 등록한 플래그 환경 변수를 확인합니다. 
  - 서버가 구동되었다면 8.3 라인의 FIND 에 찾았다는 신호를 주고 break 합니다. 
  - 아직 구동이 안되거나 실패하여 CHECK_BLUE 에 아무 값이 없다면 5초를 쉰 뒤 다시 확인합니다. 

9. 
- FIND 에 아무 값이 없다면 구동되던 green 서버는 그대로 두고 조금 전에 실행하였던 blue docker compose 파일을 중단합니다. 
- FIND 에 값이 들어가 있다면 green 서버를 내립니다.

if 로 확인할 것이 많아 다소 길지만 크게 복잡한 구문은 없었습니다. 

#### docker compose

지난 포스트까지는 docker compose 파일에 nginx 와 앱, DB 를 한번에 담고 실행하였습니다.

이 과정에서는 docker compose 파일 안의 컨테이너는 따로 네트워크를 지정하지 않는다면 자동으로 "디렉토리명_default" 이름으로 같은 네트워크를 공유합니다.

하지만 docker compose 가 여러 개로 따로 실행될 시에는 default 네트워크로 들어간다 하더라도 서로 간의 컨테이너를 인식하지 못하여 통신이 되지 않습니다.

![도커 컴포즈 아키텍쳐](/assets/images/studypot-docker-architecture.png)

이번 프로젝트에서는 그림과 같이 3개의 docker compose 를 이용합니다.

dev 파일에는 nginx 와 DB 컨테이너가, blue 와 green 파일에는 각각 앱 컨테이너가 사용됩니다.

따라서 여러 docker compose 를 쓰기 때문에 가상의 네트워크를 도커에서 생성하여 만든 네트워크로 컨테이너들을 넣어 통신을 해야 합니다.

도커 네트워크에 대해 더 자세한 내용은 [도커 네트워크](https://www.daleseo.com/docker-compose-networks/) 글을 참고해주세요!

```bash
sudo docker network create studypot
```

create 를 이용하여 studypot 이라는 네트워크를 생성하였습니다.

만든 네트워크에 대한 정보를 알아보겠습니다.

```bash
sudo docker network inspect studypot
```

![도커 네트워크 정보](/assets/images/docker-network-info.png)

Id 와 드라이버 등에 대한 정보가 있습니다. IP 에 대한 정보도 1에 있는데 이는 나중에 사용할 예정이기때문에 기억해둡니다.

이제 docker compose 에 네트워크에 대한 설정을 합니다.

- docker-compose.dev.yml
```yaml
version: "3.9"
services:
  nginx:
    image: nginx
    ports:
      - 80:80
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ...
  mysql:
    image: mysql
    ports:
    - 3306:3306
    volumes:
    - ${PWD}/mysql:/var/lib/mysql

networks:
  default:
    external:
      name: studypot
```

networks.default.external.name 경로에 위에서 생성한 네트워크를 지정합니다.

- docker-compose.blue.yml

```yaml
version: "3.9"
services:
  study-pot-api:
    image: openjdk:11
    command: "java -jar /etc/application/study-pot-api.jar"
    ports:
      - 8081:8080
    volumes:
      - /home/ec2-user/app/travis/build/study-pot-api.jar:/etc/application/study-pot-api.jar
    environment:
      - DEPLOY_NAME=blue
      - ...
networks:
  default:
    external:
      name: studypot
```

- 포트는 blue 와 green 두 개의 앱을 사용하기 때문에 8081과 8082 로 나누어 줄 예정입니다.


- CodeDeploy 로 부터 받은 jar 파일을 마운트합니다.
  

- 플래그를 세울 환경 변수로 blue 를 설정합니다.


- 네트워크는 dev 와 같이 아까 만든 네트워크를 지정합니다.

green 도 blue 파일과 유사하게 바꿀 곳만 바꾸어 만들었습니다.

이제 가장 중요한 nginx 파일을 작성하겠습니다.

#### nginx

```bash
user nginx;
worker_processes 1;

pid         /var/run/nginx.pid;

events {
    worker_connections  1024;
}
http {
    ...
    
    include /etc/nginx/conf.d/*.conf;

    upstream study-pot-app { # --- 1
        server docker-network-ip:8081 max_fails=3 fail_timeout=10s; # --- 2
	      server docker-network-ip:8082 max_fails=3 fail_timeout=10s; 
}
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  localhost;
        include /etc/nginx/default.d/*.conf;

        location / {
            proxy_pass         http://study-pot-app; # --- 3
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

1. 
- 기본적으로 http 안의 upstream 항목으로 여러 서버 그룹을 지정하고 3 의 proxy_pass 에 upstream 이름을 지정합니다. 
- Nginx 문서에 따르면 로드 밸런기본 알고리즘으로는 Round Robin 방식을 사용한다고 합니다. 이외에 least connections 방식이나 IP Hash 방식 등을 사용할 수 있습니다.
2. 
- 서버 IP 또는 도메인 이름을 지정합니다. 
- docker-network-ip 에는 아까 생성한 docker 네트워크의 Gateway 의 IP 를 지정합니다.
- 두 개의 서버를 등록하면서 두 개의 서버가 다 떠있다면 Round Robin 방식으로, 하나만 떠 있다면 떠 있는 서버로 요청을 보냅니다.
3. 
- proxy_pass 항목에는 upstream 항목의 이름을 지정합니다. 
- 주의할 점은 여기서는 포트를 적지 않습니다!

### 결과

docker compose dev 파일과 blue 파일을 먼저 실행시킵니다.

```bash
sudo docker-compose -p dev -f docker-compose.dev.yml up -d
sudo docker-compose -p app-blue -f docker-compose.blue.yml up -d
```

이제 배포를 다시 해보겠습니다. 

플래그 환경 변수를 추가한 application.yml 파일과 새로 생성한 Controller 파일을 커밋/푸쉬 합니다.

그리고 execute-deploy.sh 에서 작성했던 tempLog 를 확인합니다.

![구동 확인](/assets/images/nonstop-sh-log.png)

아까 작성하였던 로그가 찍혀 있는 것을 확인할 수 있습니다.

![네트워크 사진](/assets/images/network-container.png)

이상으로 무중단 배포까지 적용해보았습니다. 

꽤 긴 시간 할애했고 다양한 툴을 적용하여 매우 보람찬 시간이었습니다. 

참고 자료:

[nginx](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
[도커 네트워크](https://www.daleseo.com/docker-compose-networks/)