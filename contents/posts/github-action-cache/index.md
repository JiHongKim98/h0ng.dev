---
title: "GitHub Actions 캐싱처리 꼭 해야할까?"
description: GitHub Actions caching 처리
date: 2024-01-03
update: 2024-01-03
tags:
  - ci/cd
  - github actions
series: 배포
---

> Django 기준으로 구현한
> 코드는 [GitHub](https://github.com/JiHongKim98/github-action-practice/blob/main/.github/workflows/cache.yml)에서 확인 가능합니다.

## Cache를 사용해야하는 이유

GitHub Cache Action은 매 Workflow에서 build에 필요한 의존성 패키지 설치 작업을 Cache 처리해 build 속도를 향상시키기 위해 사용한다.

예를 들어, DRF를 통해 REST API를 구현하고 GitHub Action을 통해 CI를 구현했다고 했을 때
설정한 Event trigger 즉, `push` 혹은 `pull request` 이벤트가 발생할 때마다 `Runner`는 계속해서 `requirements.txt`에 작성된 의존성 패키지를 다운받아 build
완료까지 오랜 시간이 소모되고, 이로 인해 실제 test를 진행및 완료되기까지 오랜 시간이 걸리게 된다.

이를 개선하려면, 의존성 패키지 설치 작업을 캐싱처리하여 CI 테스트시 캐싱된 작업이 있는지 검사하고 있다면 해당 작업을 가져오고, 없을 때는 의존성 패키지 설치 작업을 진행 후 다시 캐싱 처리하는 방식을 사용하면
빠르게 build하고 테스트까지 진행이 가능하다.

한번 캐싱처리 전과 후의 시간 차이를 확인해보자.

![](https://velog.velcdn.com/images/kimjihong/post/3b8229de-47d8-43c7-a3f1-2358071b10ff/image.png)

Cache 하기전의 build및 test 완료까지 걸린 시간은 1분10초, 캐싱 처리 후의 소모 시간은 46초로 약 25초가량 CI 테스트 시간이 줄어들었다.
(둘다 오래걸리진 않았지만 대략 52% 의 성능 속도 향상..!)

지금은 테스트를 위해서 간단하게 구현한 것이라 캐싱처리 전에도 1분대로 엄청 느리지는 않지만, 만약 프로젝트가 커지게 된다면 그에 따라 의존성 패키지도 많아지게 되어 build 시간이 더 커지게 될 것이다.

그럼 캐싱처리는 어떻게 구현 해야할까?
<br>

## Cache 구현

먼저 Actions 탭에서 Django workflow를 선택했을 때 제공해주는 `django.yml` 파일에서 `steps` 부분을 살펴보자.

```yml
# .github/workflow/django.yml

jobs:
  # ...
  steps:
    - name: Check out repository code
      uses: actions/checkout@v3

    - name: Set up Python ${{ matrix.python-version }}
      id: setup_python
      uses: actions/setup-python@v3
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      shell: bash
      run: |
        pip install --upgrade pip
        python -m pip install -r requirements.txt

    - name: Run Tests
      run: |
        python manage.py test
```

위 처럼 매 CI test가 진행될 때마다 `requirements.txt` 파일에 작성된 의존성 패키지들을 설치하고 test를 진행한다.

이제 우리가 고려해야할 부분은 "어디를 캐싱처리 할 것인가?" 이다.

나는 의존성 패키지를 가상환경에 설치하고 해당 가상환경 자체를 Cache에 저장해 Cache가 유효할 때까지 `pip install` 작업 즉, 의존성 패키지 설치 작업을 생략하도록 구현했다.
<br>

로직은 다음과 같다.

- `IF` 캐싱된 가상환경이 있고, `requirements.txt` 파일이 변경되지 않았는가?
- `True:`
    - 해당 가상환경을 가져오고 의존성 패키지 설치 작업 **Skip**.
- `False:`
    - 가상환경 생성 및 의존성 패키지 설치 작업 **Start**.
      <br>

위 로직을 구현하기 위해 첫번째로 캐싱된 가상환경이 있는지 확인하는 `steps`을 구현했다.

```yml
- name: cache virtualenv
  uses: actions/cache@v3
  id: cache-venv # Step 식별을 위한 고유 식별자
  with:
    path: ./.venv/
    key: |
      ${{ runner.os }}-${{ steps.setup_python.outputs.python-version }}-venv-${{ hashFiles('requirements.txt') }
    restore-keys: |
      ${{ runner.os }}-${{ steps.setup_python.outputs.python-version }}-venv-
```

Runner가 실행되는 환경에서 의존성 패키지들의 정보가 작성된 `requirements.txt` 이 변경되지 않을 경우에만 캐싱된 가상환경을 가져오기 위해
Cache Key 명을 `"{실행환경}-{python-version}-venv-{해시값}"` 으로 지정하였다.

따라서 `requirements.txt` 파일이 변경되지 않았고, Cache가 유지되는 기간동안에는 Cache에서 가상환경을 가져올 수 있다.
<br>

두번째로, 만약 Cache에 일치하는 Key가 없는 경우 의존성 패키지를 설치하고
일치하는 Key가 있는 경우 의존성 패키지 설치 작업을 건너뛰는 `steps`을 구현했다.

```yml
- name: Install dependencies
  shell: bash
  run: |
    python -m venv ./.venv
    source ./.venv/bin/activate
    pip install --upgrade pip
    python -m pip install -r requirements.txt
  if: steps.cache-venv.outputs.cache-hit != 'true'
```

첫번째 구현 로직의 고유 식별자(id)인 `cache-venv` 를 통해
`if`문으로 캐시를 가져오지 못했을 경우에만 가상환경내 의존성 패키지 설치 작업을 진행하도록 구현했다.

이로써, 불필요하게 계속해서 의존성 패키지를 설치하는 과정을 **Skip** 할 수 있다.
<br>

마지막으로, Cache에 저장된 가상환경이 없는 경우 **가상환경에 진입 후** 패키지를 설치하기 때문에 문제가 없지만, 만약 Cache에 저장된 가상환경이 있을 경우 가상환경 진입과정을 건너뛰게 되므로 test
시작 전에는 꼭 가상환경에 진입하도록 구현해야한다.

```yml
- name: Run Tests
  shell: bash
  run: |
    source ./.venv/bin/activate  # 가상환경 진입 필수!
    python manage.py test
  env:
  # ...
```

### Logic Test

Cache 기능을 적용하고 다시 Action을 실행했을 때

![](https://velog.velcdn.com/images/kimjihong/post/3dfb79c3-e280-4b74-b06a-37f3c8f59d8a/image.png)

위 사진과 같이 `cache virtualenv` 작업에서 캐시를 찾지 못했다는 문구가 뜨는데
이는, 지금 새로 Cache 기능을 적용한 뒤 처음 Action을 실행하는 것이라 찾지 못한 것이다.

따라서, 의존성 패키지 설치 작업을 마친 후, CI 테스트를 진행한다.
가상환경을 Cache내에 저장하는 것은 `steps`의 모든 작업을 완료한 뒤에 이뤄진다.
(`actions/cache@v3` 의 작업)
<br>

한번, Actions의 Cache 탭을 확인해보자.

![](https://velog.velcdn.com/images/kimjihong/post/04599fe5-101b-44f2-9943-85623e6e19fc/image.png)

2개의 테스트 실행환경에서 가상환경을 캐싱하여 저장된 것을 볼 수 있다.
<br>

이제, 다시 `push` 이벤트를 발생시켜 저장된 Cache를 정상적으로 가져오는지 확인 해보자!

![](https://velog.velcdn.com/images/kimjihong/post/2125ebdb-04a7-41d9-96cd-9b2f5ce63db8/image.png)

`cache virtualenv` 작업에서 캐시를 찾아서 `Install dependencies` 작업을 건너뛰고, CI 테스트를 진행하는 것을 볼 수 있다.

이로써, Cache 적용 후 처음 Actions 실행시 진행했던 의존성 패키지 설치 단계를 건너뛰어 약 20초의 시간을 단축할 수 있게 되었다.
<br>

참고할 점은 Cache의 유효기간과 최대 용량은 GitHub 무료티어 기준으로 유효기간 7일, 최대 용량 2GB이다.
<br>

## Ref.

https://mckornfield.medium.com/caching-python-installs-in-github-actions-8309e12a15e6
