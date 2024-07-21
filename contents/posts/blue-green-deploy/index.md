---
title: 무중단 배포 일지 (Feat. Blue-Green)
description: self hosted runner 와 docker 를 사용한 blue green 무중단 배포 구현
date: 2024-06-30
update: 2024-06-30
tags:
  - ci/cd
  - github actions
series: 배포
---

> 🙇🏻 잘못된 내용 혹은 개선할 내용이 있다면 아낌 없이 피드백 주시면 정말 감사합니다! 🙇🏻

현재 카카오 클라우드 스쿨에서 진행한 커뮤니티 서비스를 실제로 사용자들이 이용할 수 있게 배포를 진행해보기로 했다.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/6a788eb3-6943-4ac0-bc46-6f664da748aa)

먼저 특별한 전략없이 일반적으로 배포하는 흐름을 알아보자.

1. 새롭게 배포할 애플리케이션 V2를 빌드한다. (ex. Dockerfile 빌드)
2. EC2 서버에 SSH 접속을 한다.
3. V2 빌드 파일을 EC2 서버로 가져온다.
4. 기존의 운영중인 V1 애플리케이션을 종료한다.
5. V2 빌드 파일을 실행한다.

<br/>

특별한 배포 전략이 없다면 보통 위와 같은 흐름을 가지게 된다.<br/>
~~(사실 위 과정은 Recreate 배포 전략이다.)~~

위 흐름에서의 문제점은, 4 - 5 번 과정에서 애플리케이션이 종료되어 사용자가 내 서비스를 이용하지 못하게 되는 시간이 존재하게 된다.

나는 이러한 문제점을 개선하기 위해, 자동화된 무중단 배포 전략을 사용하여 `DOWNTIME` 을 줄여 사용자가 현재 새로운 서비스로 Release 되고 있는지 인지하지 못하게 함으로 써 사용자 경험을 개선해보고자
한다.

> **DOWNTIME 이란?**
>
> <br/>
> 배포 과정에서 새로운 기능 혹은 수정된 기능을 제공하기 위해 이전 버전을 종료하고
> 새로운 버전이 시작되기까지 서비스를 사용할 수 없는 시간을 의미한다.
> <br/>
> <br/>
> → 즉, 사용자에게 새로운 서비스를 제공하기 위해 서비스를 이용할 수 없는 시간을 말한다.

## 배포 전략 선택

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/42b2a0ea-8245-4f92-8b38-47dc520cbf25)

먼저, 현재 내 프로젝트에 적용하기 위해 고려한 무중단 배포 전략은 총 3가지이다.

1. Rolling Update
2. Blue-Green
3. Canary Release

### Rolling Update

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/3712f892-df1d-49a9-b6c1-e00c76a590f5)

Rolling Update 배포 전략은 점진적으로 트래픽을 새로운 버전 V2 로 이전 시켜가는 무중단 배포 전략이다.

하지만, Rolling Update 배포 전략은 이전 버전인 V1 과 최신 버전인 V2 가 동시에 실행될 수 있어, 서로 다른 버전 간 충돌이 일어날 수 있어 배포시 주의가 필요하다.

따라서, Rolling Update 배포 전략을 사용하기 위해서는 서로 다른 버전간의 호환성 테스트를 꼼꼼히 테스트해야하고

신규 버전으로 배포 실패시 롤백 과정을 구현하기 어려울 것 같다는 생각에 Rolling Update 배포 전략은 선택하지 않았다.

### Blue-Green

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/e22b047f-3643-4ded-88db-b09f4bda76a4)

Blue Green 전략은 Rolling Update 전략과 달리 한번에 트래픽을 V1 에서 V2 로 옮기는 무중단 배포 전략이다.

Blue Green 전략은 버전 업데이트 실패 시 LB가 다시 V1 으로 라우팅 하면 되서 업데이트 실패에 대한 롤백 과정 구현이 상대적으로 매우 쉽다.

또한, 모든 트래픽을 V1 에서 V2 로 옮기기 때문에 버전간의 호환성 문제가 발생할 일도 사라지게 된다.

하지만, 최대 단점인 V1 과 V2 서비스가 동시에 띄우는 방법이라 시스템 리소스를 많이 먹는 문제가 있다.

### Canary Release

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/8a3f4b52-c7be-4387-8b1f-e607bd6eb786)

마지막으로 고려해본 전략은 카나리 배포 전략이다.

카나리 배포 전략은 Rolling Update 전략과 비슷하게 트래픽을 점진적으로 새로운 버전으로 이전 시켜가며 배포하는 전략이다.

하지만, 중요한 차이점은 Rolling Update 전략과 달리 <u>새로운 버전을 특정 사용자 그룹에만 배포</u>하여 문제가 없을 경우에 점진적으로 트래픽을 이전 시켜가는 비율을 증가 시킨다는 것이다.

즉, 처음 25% 의 사용자 그룹에게만 V2 버전을 배포하여 문제가 없다면 그다음 50%... 100%.. 로 점차 비율을 늘려가며 마지막엔 서비스 전체가 V2 버전으로 업데이트 되도록 하는 배포 전략이다.

> **TMI) 카나리 배포 전략을 이해하기 위한 카나리아의 유래**
>
> <br/>
> 옛날 광부들이 지나가려는 통로에 가스 누출이 되었는지 확인하기 위해, <br/>
> 유독 가스에 민감한 카나리아 새를 통해 위험을 미리 탐지 했던 방법에서 유래된 방법이다.
> <br/>
> <br/>
> 즉, 미리 소수의 사용자들에게만 배포를 진행해보고 문제가 없을 경우<br/>
> 점점 비율을 늘려가며 최종적으로 전체 서비스가 업데이트 되어가는 무중단 배포 전략을 의미한다.

### 최종적으로 선택한 배포 전략

Rolling Update 전략은 배포 실패에 대한 롤백 과정을 구현하기 까다로울 것 같다는 생각에 고려하지 않았다.

또, 카나리 배포 전략은 현재 내 서비스를 이용하는 사용자가 없기도 하고, 네트워크 관리에 대한 비용과 러닝커브가 클 것 같다는 생각으로 고려하지 않았다.

최종적으로 <u>Blue-Green 배포 전략</u>을 선택했고,<br/>
Blue 서비스와 Green 서비스가 동시에 실행하여 서버 리소스를 많이 사용하는 문제를 개선하기 위해,<br/>
업데이트가 정상적으로 마무리 될 경우 <u>이전 버전은 종료</u>하도록 구현했다.

~~(사실 현재 하나의 EC2 서버밖에 보유하지 않아 어떤 배포 전략을 선택해도 비슷하게 구현될 것 같다..)~~

### 전체 아키텍처 구성과 흐름

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/4b7db324-5b43-4f3a-8f3d-8d43ffe57599)

앞서 설명한 무중단 배포를 위해 선택한 전략인 Blue-Green 배포의 전체적인 흐름이다.

프론트엔드는 프리티어를 최대한 이용하고자 AWS 에서 제공하는 CDN 인 CloudFront 를 선택하였다.<br/>
CloudFront 를 사용하면 캐시 무효화 작업만으로 프론트엔드에서 간단하게 무중단 배포를 구현할 수 있다.

**백엔드 무중단 배포 전략의 흐름 (Blue-Green 배포)**

1. Github 메인 브랜치로 PUSH 이벤트가 발생한다.
2. Github Action 을 통해 스프링 애플리케이션 빌드 이미지를 Docker Hub 로 push 한다.
3. Self Hosted Runner 를 통해 EC2 서버에서 이미지를 pull 한다.
4. Blue 혹은 Green 컨테이너를 띄운다.
5. Health Check 에 통과하면 NGINX 설정을 교체한다. (Blue → Green 혹은 Green → Blue)
6. 기존에 운영 중인 Blue 혹은 Green 컨테이너는 종료한다. (서버 리소스 절약)

<br/>

**프론트엔드 무중단 배포 전략의 흐름**

1. Github 메인 브랜치로 PUSH 이벤트가 발생한다.
2. Github Acton 을 통해 리액트 애플리케이션을 빌드 후 S3 에 저장한다.
3. CloudFront 의 캐시를 무효화 하여 사용자가 새로운 버전의 리액트를 사용할 수 있도록 한다.

<br/>

이번 게시글에서는 백엔드에서 어떻게 Blue-Green 배포를 구현할 수 있는지 단계별로 알아보자.

## 백엔드 무중단 배포 구현

Blue-Green 무중단 배포를 구현하기 위해 사용할 환경들

- **GitHub Action**
    - 스프링 애플리케이션 빌드 및 이미지를 push


- **Self Hosted Runner**
    - EC2 서버에서 이미지는 pull 받고 배포 스크립트를 실행시킴


- **AWS EC2 - (Amazon Linux)**
    - 필자는 Amazon Linux 환경의 EC2 서버를 사용


- **Docker Hub**
    - 빌드된 스프링 이미지들을 가지고 있는 도커 리포지토리

### Dockerfile 작성

먼저, 우리가 구현한 스프링 애플리케이션을 실행하는 이미지를 만들기 위해 `DockerFile` 을 작성해주자.

```Dockerfile
# ./Dockerfile

FROM amazoncorretto:17-alpine-jdk

WORKDIR /app
# 각자의 프로젝트 파일에 맞게 변경하십셔
COPY ./build/libs/community-0.0.1-SNAPSHOT.jar /app/hong-community.jar
ENV TZ=Asia/Seoul

CMD ["java", "-jar", "hong-community.jar"]
```

### Docker Hub 리포지토리 생성

나는 이미지를 빌드하고 docker hub 리포지토리로 Push 후 EC2 서버에서 이미지를 가져와 사용할 예정이다.

그럼 docker hub [리포지토리 탭](https://hub.docker.com/repositories)에서 리포지토리를 생성해보자.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/76be542d-bde9-480e-8b1b-16d7009bff19)

이미지 빌드시 secret key 와 같이 민감한 정보와 함께 빌드될 수 있어 <u>private 리포지토리로 만드는 것</u>을 추천한다.<br/>
private 리포지토리는 docker hub 무료 플랜(Personal) 기준으로 한개만 생성 가능하다.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/f34fd03e-fc8a-40ec-83b7-4dae34836cbd)

정상적으로 생성이 완료되면 위 사진과 같이 private 리포지토리로 생성된 것을 확인할 수 있다.

### Github Action Workflow 작성 - 이미지 빌드 및 Push

이제, `.jar` 로 빌드된 스프링 애플리케이션을 실행하는 이미지를 생성하고 docker hub 로 push 하는 `docker-build-and-push` 작업을 작성해보자.

먼저, Github Action 을 동작시킬 trigger 이벤트를 설정해준다.

나는 배포 작업은 `main` 브랜치로 push 이벤트가 일어날 때 Github Action 이 동작하도록 구현했다.

```yaml
# ./.github/workflows/backend-prod-cd.yml

on:
  workflow_dispatch:
  push:
    branches:
      - main
```

그 다음으로, 스프링 애플리케이션을 `.jar` 로 빌드하는 작업을 추가해주자.

```yaml
# ./.github/workflows/backend-prod-cd.yml

# ...

jobs:
  docker-build-and-push:
    # Github Action 은 우분투 환경으로 진행
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      # JDK 설치
      - name: JDK 17 버전 설치
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      # 스프링 애플리케이션을 .jar 파일로 빌드
      - name: Gradle 을 통해 빌드
        uses: gradle/gradle-build-action@v2.6.0
        with:
          arguments: bootJar

    # ...
```

마지막으로, `.jar` 파일로 빌드한 스프링 애플리케이션을 실행할 수 있도록 Docker image 를 만들고,<br/>
EC2 서버에서 해당 이미지를 가져올 수 있도록 docker hub 로 push 하는 작업을 추가해주자.

```yaml
# ./.github/workflows/backend-prod-cd.yml

# ...
jobs:
  docker-build-and-push:
    runs-on: ubuntu-latest

    steps:
      # ...

      - name: Docker 빌드 도구 설정
        uses: docker/setup-buildx-action@v2.9.1

      - name: Docker Hub 로그인
        uses: docker/login-action@v2.2.0
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          platforms: linux/amd64
          tags: ${{ secrets.DOCKER_HUB_REPOSITORY }}:${{ secrets.IMAGE_TAG }}
```

나는 Docker Hub 에 로그인 하기 위해 `username` 및 발급받은 `access token (password)`  과<br/>
생성한 이미지를 push 할 docker hub 리포지토리와 tag 명을 Github Action 의 secrets 변수로 설정 해줬다.

→ `access token` 은 docker hub 의 [security 설정](https://hub.docker.com/settings/security)에서 발급 받을 수 있다.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/edcfddaf-e3bd-409c-8e2d-0f4e94472e9f)

`DOCKER_HUB_REPOSITORY` 는 `username/repository-name` 으로,<br/>
`IMAGE_TAG` 는 프로젝트에 맞게 지정하면 된다.

나는 `DOCKER_HUB_REPOSITORY` 을 `kimjihong/hong-community-layout` 으로 설정하고<br/>
`IMAGE_TAG` 는 `latest` 로 지정했다.

### Self Hosted Runner 설정

사실 AWS CodeDeploy 를 통해 EC2 서버에서 배포해도 되고,<br/>
Github Action SSH 를 사용하여 EC2로 접속해서 배포해도 되고 여러가지 방법 있다.

하지만, 내가 Self Hosted Runner 를 사용한 이유는 아래와 같다.

1. 배포의 전체 과정을 <u>한곳에서 몰아서 보고싶다.</u>
2. Github Action 에서 SSH 를 사용하면 무료 시간에서 차감된다.

<br/>

따라서, 배포 과정을 Github Action 으로 통일하고, Github 에서 관리하는 인스턴스가 아닌 나의 EC2 서버에서 진행할 수 있는 Self Hosted Runner 를 사용하였다.

또, Self Hosted Runner 설정은 매우매우 간단스하다. ~~(사실 이게 가장 큰 이유)~~

먼저, Github Repository 설정에서 Runners 에 들어간다.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/9083aaaf-9629-4e55-bbd2-7739307f41b6)

Runners 탭에서 `New self-hosted runner` 버튼을 누르면 아래와 같이 설정 스크립트를 친절하게 알려준다.

우리는 단순히 복붙만으로 빠르게 self hosted runner 설정을 끝낼 수 있다..!

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/df331ab1-a543-4f2d-8d91-3d68cb97dac2)

내 EC2 서버의 OS 는 Amazon Linux 이므로 Linux 버전으로 선택했다.

이제, 스크립트를 EC2 서버에서 실행하여 입맛에 맞게 설정해주자.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/7bbc2589-a0fc-4f66-a954-b9e8658ab81b)

나는 runner 를 백그라운드에서 실행시키기 위해 `nohup ./run.sh > output.log 2>&1 &` 로 실행시켰다.

정상적으로 실행이 완료되고, 다시 Runner 탭으로 들어가보면 runner 가 Idle 상태로 되어 있는 것을 확인되었다면 정상적으로 runner 가 등록된 것이다.

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/c0190706-2013-4fdc-acf8-6c64f57b54b2)

나는 여기서 추가로 러너가 장시간 켜져있을 경우 종료될 수도 있다고 판단하여<br/>
매일 새벽 3시마다 runner 를 재실행하도록 크론 잡을 설정해줬다.

```shell
# 크론 편집기 열기
$ crontab -e


# crontab 에 아래 스크립트 저장
# 매일 새벽 3시마다 runner 재실행
0 3 * * * cd /home/ec2-user/actions-runner && sudo ./svc.sh restart
```

`svc.sh` 는 self-hosted runner 에서 제공하는 스크립트로,<br/>
self-hosted runner 설치, 시작, 중지, 재시작 등의 작업을 할 수 있다.

### Blue & Green docker-compose 설정

Runner 설정이 완료 되었다면, 이제 `blue` 로 띄울 스프링 컨테이너와 `green` 으로 띄울 스프링 컨테이너를 정의 해줘야 한다.

나는 `docker-compose.blue.yml` 과 `docker-compose.green.yml` 두개의 파일로 나눠서 설정해줬다.

주의 할 점은, Runner 가 EC2로 접속하는 경로는 홈 경로가 아니라 기본적으로<br/>
`/home/ec2-user/actions-runner/_work/<repo_name>/<repo_name>/` 로 접속한다.

따라서, docker-compose 파일은 해당 디렉토리 내에 위치하도록 해야한다.

```yaml
# docker-compose.blue.yml
version: '3'

services:
  spring-backend:
    container_name: spring-backend-blue
    image: kimjihong/hong-community-layout:latest
    ports:
      - "8081:8080"
    env_file:
      - .env

---

# docker-compose.green.yml
version: '3'

services:
  spring-backend:
    container_name: spring-backend-green
    image: kimjihong/hong-community-layout:latest
    ports:
      - "8082:8080"
    env_file:
      - .env
```

`blue` 컨테이너의 호스트 포트는 8081로, `green` 컨테이너의 호스트 포트는 8082로 설정하고 컨테이너 포트는 8080 포트로 동일하게 설정해줬다.

따라서, 현재 실행중인 서비스가 `blue` 일 경우 NGINX 는 8081 포트를, 현재 실행중인 서비스가 `green` 일 경우 NGINX 는 8082 포트를 참조하도록 설정해야한다.

### NGINX 설정

NGINX 에서 배포가 성공적으로 이뤄질 때, 참조하는 스프링 컨테이너의 포트를 동적으로 변경하기 위해 `service-url.inc` 파일을 생성했다.

`service-url.inc` 파일에서 `service_url` 변수를 정의하여 서비스를 제공할 스프링 애플리케이션의 주소를 설정하고, `nginx.conf` 파일에서 이 변수 값을 `proxy_pass` 지시어의
주소로 사용되도록 구현했다.

```nginx
# /etc/nginx/conf.d/service-url.inc

set $service_url http://127.0.0.1:8081;
```

```nginx
# /etc/nginx/nginx.conf

server {
	# ...
	include /etc/nginx/conf.d/service-url.inc;

	location / {
			proxy_pass $service_url;

			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header HOST $http_host;
			proxy_set_header X-NginX-Proxy true;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header X-Request-ID $request_id;
			proxy_redirect off;
	}

	# ...
}
```

나는 현재 EC2 인스턴스에서 하나의 NGINX 만 사용하기도 하고, 네트워크 설정에 대한 복잡성 때문에 NGINX 는 docker 로 띄우지 않고, EC2 내부에 직접 설치하여 사용하였다.

### deploy.sh 작성 (blue-green 배포 스크립트)

이제, Blue-Green 배포의 핵심인 배포 스크립트를 작성해보자.

내가 작성한 배포 스크립트의 흐름은 다음과 같다.

1. 먼저 <u>현재 실행중인 서비스</u>가 무엇인지 확인한다.
2. 현재 실행중인 서비스가 blue 일 경우 <u>green 컨테이너를 새로 띄운다.</u> (green 일 경우 blue)
3. 컨테이너가 실행되기까지 지정된 시간 만큼 <u>대기</u>한다.
4. 스프링 애플리케이션에서 미리 작성한 <u>health check 엔드포인트로 요청</u>을 보낸다.
5. 200 OK 응답이 올 때까지 지정한 횟수 만큼 반복적으로 요청을 보낸다.
6. NGINX 가 참조하고 있는 `service-url.inc` 파일을 <u>green 컨테이너에 맞게 수정</u>한다.
7. `nginx -s reload` 로 NGINX 중단 없이 `.conf` 파일만 <u>다시 갱신</u>한다.
8. 기존에 실행중이였던 서비스 <u>blue 컨테이너를 종료</u>한다.

<br/>

위와 같은 흐름을 가지며 스크립트 실행 도중 새로운 버전 즉, green 컨테이너를 띄우는 과정에서 오류가 발생한다면 신속 blue 컨테이너로 다시 롤백하는 흐름을 가진다.

아래는 위 흐름을 스크립트로 작성한 파일이다.

```shell
#!/bin/bash

# hong: 리팩토링 해놔서 아래 변수만 설정에 맞게  변경하셔서 쓰심 됨다

# 설정 변수
CONTAINER_NAME="spring-backend"  # 컨테이너 prefix
CONTAINER_SETUP_DELAY_SECOND=10  # 컨테이너 실행 지연 시간
MAX_RETRY_COUNT=15  # 서버 상태 확인 최대 시도 횟수
RETRY_DELAY_SECOND=2  # 서버 상태 확인 지연 시간(초)
BLUE_SERVER_URL="http://127.0.0.1:8081"  # blue 서버 URL
GREEN_SERVER_URL="http://127.0.0.1:8082"  # green 서버 URL
HEALTH_END_POINT="/api/health"  # 서버 health check 를 위한 엔드포인트 (200 응답만 오면 됩니당)
BLUE_DOCKER_COMPOSE_FILE_NAME="docker-compose.blue"  # blue 의 docker-compose 파일명 (ex. `docker-compose.blue.yml`)
GREEN_DOCKER_COMPOSE_FILE_NAME="docker-compose.green"  # green 의 docker-compose 파일명 (ex. `docker-compose.green.yml`)
NGINX_SERVICE_URL_FILE="/etc/nginx/conf.d/service-url.inc"  # NGINX 설정 파일 경로

# NGINX 재로드 함수
reload_nginx() {
    echo "NGINX 설정 변경 작업 시작"

    if nginx -t; then
        nginx -s reload
        echo "NGINX 설정 재로드 완료"
    else
        echo "NGINX 설정 오류 -> 롤백 수행"
        echo "set \$service_url $CURRENT_SERVICE_URL;" > $NGINX_SERVICE_URL_FILE
        nginx -s reload
        exit 1
    fi
}

# 헬스 체크 함수
health_check() {
    local REQUEST_URL=$1
    local RETRY_COUNT=0

    while [ $RETRY_COUNT -lt $MAX_RETRY_COUNT ]; do
        echo "상태 검사 ( $REQUEST_URL )  ...  $(( RETRY_COUNT + 1 ))"
        sleep $RETRY_DELAY_SECOND

        REQUEST=$(curl -o /dev/null -s -w "%{http_code}\n" $REQUEST_URL)
        if [ "$REQUEST" -eq 200 ]; then
            echo "상태 검사 성공"
            return 0
        fi

        RETRY_COUNT=$(( RETRY_COUNT + 1 ))
    done

    return 1
}

# 컨테이너 시작 함수
start_container() {
    local COLOR=$1
    local DOCKER_COMPOSE_FILE_NAME=$2
    local SERVER_URL=$3

    echo "$COLOR 컨테이너를 띄우는 중"
    docker-compose -p ${CONTAINER_NAME}-$COLOR -f ${DOCKER_COMPOSE_FILE_NAME}.yml up -d
    echo "${CONTAINER_SETUP_DELAY_SECOND}초 대기"
    sleep $CONTAINER_SETUP_DELAY_SECOND

    echo "$COLOR 서버 상태 확인 시작"
    if ! health_check "$SERVER_URL$HEALTH_END_POINT"; then
        echo "$COLOR 배포 실패"
        echo "$COLOR 컨테이너 정리"
        docker-compose -p ${CONTAINER_NAME}-$COLOR -f ${DOCKER_COMPOSE_FILE_NAME}.yml down
        exit 1
    else
        echo "$COLOR 배포 성공"
        echo "set \$service_url $SERVER_URL;" > $NGINX_SERVICE_URL_FILE
        reload_nginx
        echo "기존 ${OTHER_COLOR} 컨테이너 정리"
        docker-compose -p ${CONTAINER_NAME}-${OTHER_COLOR} -f ${OTHER_DOCKER_COMPOSE_FILE_NAME}.yml down
    fi
}

# 메인 스크립트 로직
if [ "$(docker ps -q -f name=${CONTAINER_NAME}-blue)" ]; then
    echo "blue >> green"
    OTHER_COLOR="blue"
    OTHER_DOCKER_COMPOSE_FILE_NAME=$BLUE_DOCKER_COMPOSE_FILE_NAME
    start_container "green" $GREEN_DOCKER_COMPOSE_FILE_NAME $GREEN_SERVER_URL
else
    echo "green >> blue"
    OTHER_COLOR="green"
    OTHER_DOCKER_COMPOSE_FILE_NAME=$GREEN_DOCKER_COMPOSE_FILE_NAME
    start_container "blue" $BLUE_DOCKER_COMPOSE_FILE_NAME $BLUE_SERVER_URL
fi
```

~~(저와 동일하게 진행하고 있으시다면, 아래의 스크립트의 상단에 환경 변수만 프로젝트에 맞게 수정해서 사용하시면 됩니다!)~~

### Github Action Workflow 작성 - 이미지 Pull 및 실행

EC2 내부에서 Blue-Green 배포를 위한 설정들을 마쳤다면,<br/>
마지막으로 Runner 가 EC2에서 진행할 작업을 정의해줘야한다.

```yaml
# ...

jobs:
  docker-build-and-push:
  # ...

  docker-pull-and-run:
    # 우리가 설정한 runner 의 실행 환경
    runs-on: [ self-hosted, Linux, X64 ]
    needs: [ docker-build-and-push ]
    # 도커 이미지 빌드 및 리포지토리로 push 가 성공할 경우에만 진행
    if: ${{ needs.docker-build-and-push.result == 'success' }}

    steps:
      # 프로젝트에서 환경 변수를 .env 로 관리한다면 동일하게 진행하심 됩니다.
      # 사용하지 않는다면 지워주시면 됩니다.
      - name: 환경변수 파일 생성
        env:
          PROPERTIES_PROD: ${{ secrets.PROPERTIES_PROD }}
        run: |
          touch .env
          echo "${PROPERTIES_PROD}" > .env

      # docker hub 의 private 리포지토리에 있는 이미지를 pull
      - name: Docker Hub 리포지토리에서 최신 이미지 가져오기
        run: |
          sudo docker login --username ${{ secrets.DOCKER_HUB_USERNAME }} --password ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
          sudo docker pull ${{ secrets.DOCKER_HUB_REPOSITORY }}:${{ secrets.IMAGE_TAG }}

      # deploy.sh 의 실행 권한 주기 및 실행
      - name: blue green 배포
        run: |
          sudo chmod +x deploy.sh
          sudo ./deploy.sh
```

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/63e4c924-133d-40ec-a009-78cac6019ce3)

`runs-on` 은 우리가 이전에 설정했던 self hosted runner 의 설정에 맞게 추가해주면 된다.<br/>
필자는 `self-hosted`, `Linux`, `X64` 라벨을 추가해줬다.

또, 나는 스프링 애플리케이션 실행시 필요한 환경 변수들이 있어, `.env` 파일을 EC2 서버 내부에 생성하도록 구현을 해놨다.<br/>
~~(`.gitmodule` 을 사용하여 application.yml 파일을 관리하고 이미지 빌드시 포함하는 것을 추천한다.)~~

### Blue-Green 배포 테스트

![](https://github.com/JiHongKim98/hong.dev/assets/144337839/1e0747f6-a32d-47d7-9f79-66dd12f36365)

현재 서비스중인 컨테이너가 `blue` 일 경우 `green` 으로, `green` 일 경우 `blue` 로 전환이 잘 되는 것을 확인할 수 있다.

## Ref.

https://hudi.blog/zero-downtime-deployment/<br/>
https://velog.io/@mminjg/Github-Actions-CodeDeploy를-이용한-EC2-배포-자동화<br/>
https://velog.io/@hooni_/Blue-Green-무중단-배포<br/>
