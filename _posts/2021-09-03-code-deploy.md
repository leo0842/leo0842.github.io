---
layout: single 
title: "[Dev] CI/CD 적용하기 - 2. 배포 자동화"
categories: [CodeDeploy]
tags: [Dev, CodeDeploy, AWS]

date: 2021-09-03 
last_modified_at: 2021-09-06
---

안녕하세요. 이번 게시글에서는 _CI/CD 적용하기_ 시리즈 두 번째 파트, 배포 자동화에 대해서 알아보겠습니다.

_CI/CD 적용하기_ 시리즈 첫 번째 파트, [CI/CD 적용하기 - 1. 빌드 및 테스트 자동화](https://leo0842.github.io/travis/CI-CD/) 에 이어서 작성합니다.

> 개발 환경
> - Spring Boot 2.4.5
> - Gradle 6.8.3
> - Travis CI
> - AWS EC2, IAM, S3, Role, CodeDeploy

배포 자동화를 하기 전에는 변경 사항이 있을 시,

1. ./gradlew build 로 빌드 및 jar 파일 생성
2. root directory 로 jar 파일 이동
3. sftp 를 이용하여 jar 파일을 ec2 배포 서버에 이동
4. ec2 배포 서버에 접속하여 서버를 내린 뒤 변경된 jar 파일로 다시 서버 구동

위와 같은 과정을 수동으로 직접 하였습니다.

그렇게 길지 않은 작업이지만 작은 실수가 있을 수도 있고, 사소한 변경 사항에 대해 같은 과정을 계속 해나간다는게 여간 귀찮은 작업이 아닐 수 없었습니다.

따라서 이러한 과정을 자동화 하는 작업을 진행해보았습니다.

### AWS IAM 사용자 생성

먼저 Travis CI 가 사용할 CodeDeploy 용 IAM 계정을 생성하겠습니다.

AWS 는 루트 사용자가 모든 권한을 제어하는 대신 IAM 이라는 사용자를 만들어 IAM 이 다양한 역할을 수행하도록 만들어줍니다.

IAM 에 관한 자세한 내용을 알고싶으신 분들은 [IAM 사용자 인증](https://dreamlog.tistory.com/592) 을 참고해주세요!

서비스 -> 보안, 자격 증명 및 규정 준수 카테고리에 있는 IAM 으로 들어간 뒤 사용자 탭 -> 사용자 추가를 클릭합니다.

![IAM 사용자 추가](/assets/images/codedeploy/aws-web/iam-사용자추가.png)

사용자 이름에는 자유롭게 작성한 뒤 액세스 키 ID 와 시크릿 키로 접근을 허용하는 프로그래밍 방식 액세스에 체크를 한 뒤 다음으로 넘어갑니다.

![IAM 사용자 이름](/assets/images/codedeploy/aws-web/iam-Username.png)

![IAM 두 번째 1](/assets/images/codedeploy/aws-web/iam-S3Access.png)

생성될 사용자의 권한을 설정하는 페이지입니다.

기존 정책 직접 연결을 클릭한 뒤 AmazonS3FullAccess 를 체크합니다.

AmazonS3FullAccess 는 S3 접근 권한 정책입니다.

![IAM 두 번째 2](/assets/images/codedeploy/aws-web/iam-CodeDeployAccess.png)

두 번째로 AWSCodeDeployFullAccess 를 체크합니다.

AWSCodeDeployFullAccess 는 사진에서 처럼 4개의 서비스에 대해 권한을 설정합니다.

![IAM 세 번째 태그](/assets/images/codedeploy/aws-web/iam-태그.png)

IAM 사용자에 대해 태그를 설정하는 페이지입니다. 선택사항이기에 필요없다면 그냥 넘어갑니다.

![IAM 네 번째 검토](/assets/images/codedeploy/aws-web/iam-검토.png)

검토 페이지입니다. 방금 선택했던 두 가지 정책이 다 선택되었는지 확인한 뒤 사용자 만들기를 클릭합니다.

![IAM 추가 완료](/assets/images/codedeploy/aws-web/iam-추가완료.png)

IAM 사용자 생성이 완료되었습니다. IAM 접근을 위한 액세스 키와 시크릿 키입니다. csv 다운로드를 통하여 저장합니다.

이제 해야할 일은 IAM 사용자 권한 정책에서 추가하였던 S3 와 CodeDeploy 를 생성하는 것입니다.

### S3 버킷 생성

![버킷 만들기](/assets/images/codedeploy/aws-web/s3-버킷만들기.png)

서비스 -> 스토리지 카테고리에서 S3 로 들어간 뒤 버킷 만들기를 클릭합니다.

![버킷 이름](/assets/images/codedeploy/aws-web/s3-버킷이름.png)

버킷 이름을 자유롭게 작성하고 지역도 설정합니다.

[버킷 이름 지정 규칙 참조](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html) 이런 것도 있으니 참고해보시면 좋을 것 같습니다.

![퍼블릭 액세스](/assets/images/codedeploy/aws-web/s3-퍼블릭액세스.png)

퍼블릭 액세스는 허용하였습니다.

객체에 대해 보안적으로 취약하지만 저장할 객체가 깃허브 퍼블릭 레포지토리에 푸쉬된 코드의 jar 파일인 점인 것을 감안하여 퍼블릭 액세스로 하였습니다.

퍼블릭 액세스를 차단하고 Travis CI 이용 시 접근 권한이 없어서 에러가 발생하기 때문에 따로 접근 권한을 추가해야 합니다.

이외의 버킷 버전 관리, 태그, 기본 암호화는 손대지 않고 버킷을 생성하였습니다.

### IAM Role 생성

###### 이번에는 사용자를 대신에 access key & secret key를 사용해 원하는 기능을 진행하게할 AWS Role (이하 역할)을 생성하겠습니다.

![역할 만들기](/assets/images/codedeploy/aws-web/role-역할만들기.png)

서비스 -> 보안, 자격 증명 및 규정 준수 카테고리에 있는 IAM 으로 들어간 뒤 좌측의 역할 탭에 들어가 역할 만들기를 클릭합니다.

![역할 ec2](/assets/images/codedeploy/aws-web/role-EC2선택.png)

다양한 역할 중 EC2 를 선택한 뒤 아래에서도 EC2 를 선택하고 다음으로 넘어갑니다.

![역할 ec2 정책](/assets/images/codedeploy/aws-web/role-ec2roleforcodedeploy.png)

AmazonEC2RoleforAWSCodeDeploy 를 선택하고 다음으로 넘어갑니다.

![역할 이름](/assets/images/codedeploy/aws-web/role-역할이름.png)

태그는 역시 선택사항이기에 넘어가고 이름을 작성합니다.

서비스명-EC2CodeDeployRole 으로 작성하였습니다.

![역할 codedeploy](/assets/images/codedeploy/aws-web/role-codedeploy선택.png)

두번째 역할은 CodeDeploy 로 만들겠습니다.

![역할 codedeploy 정책](/assets/images/codedeploy/aws-web/role-codedeploy정책.png)

CodeDeploy 정책에는 하나밖에 없기때문에 별다른 선택 없이 다음으로 넘어갑니다.

![역할 codedeploy 이름](/assets/images/codedeploy/aws-web/role-역할이름2.png)

서비스명-CodeDeployRole 으로 작성하였습니다.

### CodeDeploy 어플리케이션 생성

![애플리케이션 생성 창](/assets/images/codedeploy/aws-web/codedeploy-앱-창.png)

서비스 -> 개발자도구 CodeDeploy, 좌측 탭 애플리케이션 -> 애플리케이션 생성을 클릭합니다.

![애플리케이션 생성](/assets/images/codedeploy/aws-web/codedeploy-앱-생성.png)

이름을 작성하고 컴퓨팅 플랫폼으로 EC2/온프레미스 선택 후 생성합니다.

생성하면 배포 그룹에 아직 아무 것도 없습니다. 그래서 배포 그룹을 생성하겠습니다.

![첫 번째 배포 그룹](/assets/images/codedeploy/aws-web/배포그룹-이름.png)

그룹 이름은 자유롭게 작성하고 서비스 역할은 위에서 만든 CodeDeployRole 선택, 배포 유형은 현재 위치를 선택하겠습니다. 블루/그린 배포는 인스턴스 두 개가 필요합니다.

![두 번째 배포 그룹](/assets/images/codedeploy/aws-web/배포그룹-환경-구성.png)

Amazon EC2 인스턴스에서 키 값에는 Name, 밸류 값에는 배포할 인스턴스를 선택합니다.

![세 번쨰 배포 그룹](/assets/images/codedeploy/aws-web/배포그룹-기타.png)

CodeDeploy 에이전트 설치는 하지 않고 배포 구성으로는 OneAtATime 으로 설정합니다. 그리고 로드밸런서도 비활성화를 한 뒤 배포 그룹을 생성합니다.

![EC2 화면](/assets/images/codedeploy/aws-web/인스턴스-IAM-역할-수정-확인.png)

서비스 -> EC2 로 들어간 뒤 적용할 인스턴스를 체크하고 작업 -> 보안 -> IAM 역할 수정을 클릭합니다.

![IAM 역할 수정](/assets/images/codedeploy/aws-web/인스턴스-IAM-역할-수정.png)

그러면 화면과 같이 IAM 역할을 수정할 수 있는 칸이 있고 앞서 만들어 놓은 EC2CodeDeploy 역할을 클릭한 뒤 저장을 클릭합니다.

많은 부분을 만들어서 다소 복잡하였지만 이상으로 AWS 에서 할 작업은 끝이 났습니다.

이제 방금 AWS 에서 작업하였던 IAM 사용자를 EC2 에 추가하고 배포시 자동으로 서버를 내리고 다시 띄울 수 있는 쉘 스크립트를 작성하겠습니다.

### EC2

먼저 EC2 에 대하여 앞서 만들었던 IAM 사용자를 설정하겠습니다.

```bash
sudo aws configure
```

![aws configure](/assets/images/codedeploy/ec2/aws-configure.png)
 
ACCESS KEY ID 와 ACCESS SECRET KEY 에 IAM 생성 시 받았던 키를 입력하고 지역과 포맷도 각각 입력합니다.

```bash
sudo mkdir -p /home/ec2-user/app/travis/build
```

CodeDeploy 로부터 배포 받을 파일들을 저장할 디렉토리를 생성합니다.

이제 받은 jar 파일로 서버를 자동으로 실행할 쉘 스크립트를 작성하겠습니다.

```bash
sudo vim /home/ec2-user/app/travis/deploy.sh
```

```shell
#!/bin/bash

REPOSITORY=/home/ec2-user/app/travis/build

JAR_FILE=$(ls $REPOSITORY/ | grep 'study-pot-api.jar')

echo "$REPOSITORY"

if [ -z $JAR_FILE ]; then
	echo "no jar file"
else
	echo "cp $REPOSITORY/$JAR_FILE /home/ec2-user/study-pot/application/study-pot-api.jar"
	cp $REPOSITORY/$JAR_FILE /home/ec2-user/study-pot/temp/study-pot-api.jar
	echo "cd /home/ec2-user/study-pot"
	cd /home/ec2-user/study-pot
	echo "sudo docker-compose down"
	sudo docker-compose down
	sleep 10
	echo "sudo docker-compose up -d"
	sudo docker-compose up -d
	sleep 3
fi

echo "FINISH"
```

- 먼저 CodeDeploy 를 통해 전달된 디렉토리에서 jar 파일명을 가져옵니다.


- 해당 jar 파일을 기존에 저장하던 디렉토리로 복사하고 docker-compose 파일이 있는 디렉토리로 이동합니다.


- 그리고 서버를 내렸다가 다시 구동합니다.

쉘 스크립트가 있는 디렉토리로 이동하여 파일 실행 권한을 추가해줍니다.

```bash
cd /home/ec2-user/app/travis

chmod +x deploy.sh
```

그리고 테스트를 위해 레포지토리 디렉토리에 sftp 로 jar 파일을 넣고 쉘 스크립트가 정상 작동하는지 확인해보겠습니다.

```bash
./travis.sh
```

![작동하는 사진](/assets/images/codedeploy/ec2/ec2-수동쉘스크립트-구동.png)

sudo docker logs [컨테이너명] 을 통해 새로 구동되었는지 확인합니다.

![8080포트 사진](/assets/images/codedeploy/ec2/ec2-수동쉘스크립트구동도커로그확인.png)

8080 포트가 새로 뜬 것을 확인할 수 있습니다.

### 프로젝트에 파일 생성

프로젝트로 돌아와 travis 파일을 수정하고 appspec.yml 파일과 execute-deploy.sh 파일을 추가합니다.

```bash
touch ./appspec.yml ./execute-deploy.sh
```

#### .travis.yml

먼저 .travis.yml 파일에 다음과 같이 추가하였습니다.

```yaml
before_deploy:
  - cp ./build/libs/back-0.0.1-SNAPSHOT.jar ./study-pot-api.jar
  - zip -r -j jar ./study-pot-api.jar ./appspec.yml ./execute-deploy.sh
  - mkdir -p deploy
  - mv ./jar.zip ./deploy/jar.zip
```

### before_deploy

deploy 사이클을 실행하기 전 수행되는 사이클입니다.

deploy 할 때 전달 할 객체를 만듭니다.

- zip: 
  * CodeDeploy 에서 EC2 에 배포 시 jar 파일 단독으로는 지원을 하지 않습니다. 
  * 그래서 zip 을 통해 압축 파일을 만들어줍니다.


- appspec.yml: 
  * CodeDeploy 가 인식하는 설정 파일입니다.
  * Travis 설정 파일과 함께 묶어서 전달되게 합니다.


- execute-deploy.sh: 
  * CodeDeploy 단독으로는 EC2의 쉘 파일을 실행시키지 못합니다. 
  * 따라서 위의 파일들과 함께 묶은 뒤 execute-deploy 쉘을 실행시켜 EC2 의 쉘 파일을 실행시킬 수 있도록 합니다.

```yaml
deploy:
  - provider: s3
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY 
    bucket: studypot-jar-bucket
    region: ap-northeast-2
    skip_cleanup: true
    acl: public_read
    local_dir: deploy
    wait_until_deployed: true
    on:
      repo: leo0842/StudyPot
      branch: travis-ci

  - provider: codedeploy
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY 
    bucket: study-pot-jar-bucket 
    key: jar.zip 
    bundle_type: zip
    application: studypot 
    deployment_group: study-pot-group 
    region: ap-northeast-2
    wait_until_deployed: true
    on:
      repo: leo0842/StudyPot
      branch: travis-ci

```

#### deploy - S3

위에서 압축한 파일을 S3에 저장합니다.

- access_key_id, secret_access_key:
  * IAM 사용자 생성시 다운받았던 csv 파일에 있습니다.
  * 환경 변수로 작성하여 레포지토리에 올려도 외부에 노출되지 않게 합니다.
  * Travis CI 페이지에서 환경변수로 등록할 수 있습니다.


- skip_cleanup:
  * deploy 전, 워킹 디렉토리를 비우기때문에 skip 을 true 로 합니다.
  * Travis 의 View config 를 확인하면 dpl 버전 2에서 해당 변수는 deprecated key 이기때문에 'cleanup' 으로 사용하라고 하지만 변경하면 _`chdir': No such file or directory @ dir_chdir - deploy (Errno::ENOENT)_ 라는 에러가 발생합니다.
  * cleanup: false, true 로도 해보고 해당 변수를 없애도 봤지만 같은 에러가 발생합니다. 따라서 View config 의 경고는 무시하는게 나은 듯 합니다.
  

- local_dir:
  * 전체 프로젝트가 아닌 특정 폴더(해당 프로젝트에서는 before_deploy 에서 만든 deploy 폴더)만 S3 에 업로드 할 수 있게 하는 변수입니다.
  
#### deploy - CodeDeploy

S3에 저장된 파일을 EC2 배포 서버로 전달합니다.

- key: 
  * 버킷에 압축된 형태(bundled artifacts)로 저장된 파일 이름입니다.
  

- bundle_type: 
  * 전달할 파일의 확장자를 명시합니다. 
  * 사용 가능한 확장자로는 tar | tgz | zip | YAML | JSON 이 있습니다.
  
application 과 deployment_group 에는 AWS 에서 생성하였던 애플리케이션 이름과 배포 그룹 이름을 작성합니다.

#### appspec.yml

appspec.yml 은 CodeDeploy 의 설정 파일입니다.

```bash
vim ./appspec.yml
```

```yaml
version: 0.0
os: linux
files:
  - source:  /
    destination: /home/ec2-user/app/travis/build/

hooks:
  AfterInstall: 
    - location: ./execute-deploy.sh
      timeout: 180
      runas: ec2-user
```

- files.destination: 
  * EC2 배포 서버에 전달할 객체의 저장 위치입니다. 조금 전에 AWS 에서 만들었던 디렉토리입니다.


- hooks: 
  * CodeDeploy AppSpec 수명 주기 이벤트 후크입니다.
  * 수명 주기에 대해 더욱 자세한 내용은 [수명 주기 이벤트 후크 목록](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#appspec-hooks-server) 을 참고해주세요!
  * location: 실행할 파일의 위치입니다.
  
#### execute-deploy.sh

CodeDeploy 단독으로는 EC2 배포 서버의 deploy.sh 을 실행시키지 못합니다.

따라서 execute-deploy.sh 파일을 함께 보내 deploy.sh 을 실행시킬 수 있도록 합니다.

```bash
vim ./execute-deploy.sh
```

```bash
#!/bin/bash

/home/ec2-user/app/travis/deploy.sh > ~/tempLog 2> ~/errorLog
```

조금 전에 만들었던 EC2 배포 서버의 쉘 파일을 실행하게 합니다.

그리고 찍히는 로그들을 루트 디렉토리에 저장하겠습니다.

### Travis CI 환경변수 등록

이제 AWS KEY 환경 변수를 등록하러 Travis CI 페이지로 접속합니다.

![대쉬보드-레포 세팅](/assets/images/codedeploy/travis/travis-레포-설정.png)

상단 Dashboard 를 클릭 후 레포지토리 세팅에 들어갑니다.

![키 등록](/assets/images/codedeploy/travis/travis-환경-변수.png)

아래 쪽으로 스크롤하면 AWS_ACCESS_KEY 와 AWS_SECRET_KEY 등록할 수 있는 환경 변수 설정 칸이 있습니다.

BRANCH 속성에서 branch 별 환경변수 등록도 가능합니다.

### 배포 확인

모든 설정과 코드 작성 작업을 끝마쳤습니다.

이제 배포와 쉘 스크립트가 자동으로 작동 되는지 확인해보겠습니다.

지금까지 작성한 travis, appspec YAML 파일과 execute-deploy 쉘 스크립트를 커밋/푸쉬 하고 Travis CI 페이지로 접속합니다.

![배포 성공](/assets/images/codedeploy/travis/travis-배포성공.png)

_Done. Your build exited with 0._ 문구와 함께 성공한 것을 확인할 수 있습니다.

AWS 의 버킷과 CodeDeploy 의 배포 그룹을 확인해 보겠습니다.

![bucket](/assets/images/codedeploy/aws-web/s3-버킷배포확인.png)

jar 압축 파일이 잘 들어와 있습니다.

![배포 내역](/assets/images/codedeploy/aws-web/codedeploy-배포%20성공.png)

배포 내역에도 배포가 성공한 것을 확인할 수 있습니다.

마지막으로 EC2 서버에 배포가 됐는지, 쉘 스크립트가 실행이 되었는지 확인해 보겠습니다.

![build 폴더](/assets/images/codedeploy/ec2/ec2-build폴더확인.png)

S3 의 jar 압축 파일이 잘 전달된 것을 확인할 수 있습니다.

압축된 파일들이 풀린 상태로 배포가 되었습니다.

![tempLog, errorLog](/assets/images/codedeploy/ec2/ec2-tempLog-errorLog.png)

로그 파일이 생성이 되었고 echo 한 로그는 tempLog 에, 도커 로그는 errorLog 로 인식되어 errorLog 에 들어가 있습니다.

모든 작업이 정상적으로 작동하였습니다!

많은 작업을 자동으로 만들었지만 아직 중요한 작업이 남아있습니다.

docker-compose 로 서버를 내렸다가 띄우는 작업은 내렸다가 띄우는 과정에서 약 30초 정도의 서비스 중단이 발생합니다.

다음 게시글에서는 Nginx 를 활용하여 무중단으로 배포가 이루어지게끔 만들어 보겠습니다.

### 참고 자료

- [6) 스프링부트로 웹 서비스 출시하기 - 6. TravisCI & AWS CodeDeploy로 배포 자동화 구축하기](https://jojoldu.tistory.com/265)


- [AWS-Docs AppSpec 파일 구조](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/reference-appspec-file-structure.html)


- [Travis CI Documentation](https://docs.travis-ci.com/)