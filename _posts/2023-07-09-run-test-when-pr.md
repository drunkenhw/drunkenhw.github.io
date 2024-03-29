---
title:  "Pull Request 시 자동으로 테스트 실행하기"
excerpt: "Pull Request 시 자동으로 테스트 실행하기"

categories:
  - java
tags:
  - github
  - github actions
  - ci
last_modified_at: 2023-07-09T19:14:00-05:00
---
## Pull Request시 자동으로 test를 실행하면 좋은 점
pull request 생성 시 자동으로 테스트를 돌려준다면 다른 팀원의 pr을 굳이 제 로컬에 clone하여 테스트를 돌려보지 않아도 됩니다.
많은 시간을 단축할 수 있습니다.

그리고 test가 실패한다면 강제로 Merge가 되지 않도록 한다면 실수로 테스트가 되지 않는 커밋을 올리는 것을 방지할 수 있겠죠.

이 두가지만으로도 생산성이 많이 올라갈 것을 기대할 수 있습니다.

## 어떻게 할 수 있나요

Github Action을 이용하여 설정한 조건에 맞는 상황에서 명령어를 실행하여 test를 할 수 있습니다.

###  Github Action 파일 생성

1. 먼저 최상위 폴더에 `.github/workflows` 폴더를 생성합니다.
2. 해당 폴더 내에 `example.yml`을 생성합니다.
3. 아래와 같이 yml 파일을 작성합니다.

```yml
name: pr test

on:
  pull_request:
    branches:
      - main
      - develop

permissions:
  contents: read

jobs:
  test:
    name: merge-test
    runs-on: ubuntu-latest
    environment: test
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Test with Gradle
        run: ./gradlew build
```

### Job 이름 설정
복잡하지 않습니다. 먼저 **name** 속성은 github action에서 보여질 Job의 이름을 정하는 부분입니다.

지금은 `pr test`로 해두었습니다. 그럼 아래 사진과 같이 반영됩니다.

![workflows name](https://github.com/car-ffeine/car-ffeine.github.io/assets/106640954/28494d8e-66b5-4eec-a98a-414968b03306)

### workflow 트리거 설정
다음으론 `on` 속성입니다. 이 속성은 workflow를 실행할 이벤트를 지정하는데 사용됩니다. 특정 이벤트 유형과 조건을 기반으로 workflow를 트리거하도록 구성할 수 있습니다.

예를 들어 아래와 같이 정의했습니다.
```yml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - develop
```
그렇다면 이 workflow가 작동되는 시점은 `main` 브랜치에 **push**가 되거나 `develop` 브랜치에 **pull request**를 보낼 때 작동합니다.

### 권한 부여
```yml
permissions:
  contents: read
```
이런 권한을 주게 된다면 이 job은 읽기 권한밖에 없기 때문에 실수로 다른 것을 추가하지 못하게 막을 수 있습니다

### 동작할 명령어 입력
```yml
jobs:
  test:
    name: merge-test
    runs-on: ubuntu-latest
    environment: test
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Test with Gradle
        run: ./gradlew build
```

#### name
제일 간단히 볼 수 있는 **name** 설정은 아래 사진처럼 어떤식으로 보여줄지 정할 수 있습니다.

![job image](https://github.com/car-ffeine/car-ffeine.github.io/assets/106640954/43ebf3c1-4632-447f-89c3-0e74ed01dc3c)

#### runs-on
**runs-on** 속성입니다. 해당 운영체제를 사용한다고 정의하는 부분입니다. 지금은 저희가 사용할 ec2와 같은 환경인 `ubuntu`에서 작동하도록 설정했지만,
`windows-latest`, `macos-latest`로 변경할 수도 있습니다.

#### environment
**environment** 속성입니다. 해당 속성은 꼭 필요한 부분이 아니지만 branch의 rule 설정에 사용할 수 있습니다. 그리고 환경을 한꺼번에 관리할 수 있습니다.

이 부분은 아래에 branch rule을 정하는 부분을 보시면 아마 이해가 될 것 입니다.

#### defaults
해당 속성은 어떤 폴더에서 명령어를 실행할 지 지정합니다. 지금의 저희 프로젝트에서는 한 repository에 backend, frontend 폴더를 나누었기 때문에 backend 폴더로 이동하여 명령어를 실행해야 합니다.

그래서 **working-directory**를 `./backend`라고 지정했습니다.

#### steps

제일 중요한 **steps**입니다. 해당 속성은 어떤 명령어를 어떤 순서로 실행시킬지 정의합니다. 지금의 workflow에선

1. Java 17 설치
2. gradlew 파일에 실행 권한 부여
3. gradle build 실행

순으로 동작합니다.

### 다른 조건과 이벤트도 추가하고 싶어요

저희 프로젝트는 하나의 repository에서 frontend, backend 코드를 같이 관리하는 상황입니다. 하지만 frontend 코드를 수정했다고 java 테스트를 돌리는 것은 오히려 생산성이 줄어들겠죠.

그리고 frontend도 테스트를 돌리고 싶지만 gradle을 사용하지 않습니다.

그럴 때 간단한 속성을 추가하면 **파일의 변경에 따라 해당 job을 실행할 조건**을 정의할 수 있습니다.

```yml
on:
  pull_request:
    branches:
      - main
      - develop
    paths:
      - backend/**
      - .github/**
```
위와 달리 지금 **pull request**에는 속성이 하나가 더 있는데요. **paths**를 적용하면 `backend` 폴더 하위의 무언가 변경이 있는 **pull request**에만 작동을 하게 됩니다.

그럼 backend의 workflow 파일에 **paths** 속성을 하나 추가하고, 비슷한 frontend workflow를 만들어주면 되겠죠.
```yml
name: frontend test

on:
  pull_request:
    branches:
      - main
      - develop
    paths:
      - frontend/**

permissions:
  contents: read

jobs:
  test:
    name: jest
    runs-on: ubuntu-latest
    environment: test
    defaults:
      run:
        working-directory: ./frontend
    steps:
      - uses: actions/checkout@v3
      - name: NPM Install
        run: npm i
      - name: Jest run
        run: npm run test
```
이런 식으로 yml 파일을 하나 추가하면 **frontend의 수정이 일어날 때는 jest**를 실행하고, **backend 폴더의 수정이 일어나면 gradlew**를 실행하게 할 수 있습니다.

## Test가 실패하는 PR은 Merge 막기

Test가 실패하는 Pull Request가 Merge 되는 일은 절대로 없어야 합니다. 그런 실수를 방지하려면 팀원 전부가 리뷰할 때 테스트를 돌려봐야하는 귀찮음이 생길 수 있습니다.

그리고 사람은 실수해도 기계는 거짓말을 하지 않습니다. 자동으로 막도록 동작하게 만들어놓으면 그럴 일을 미연에 방지할 수 있습니다.

### Environments 확인하기
먼저 해당 Repository의 Settings -> Environments 탭으로 들어갑니다.
![environments](https://github.com/car-ffeine/car-ffeine.github.io/assets/106640954/4e3e867a-1037-46bc-865a-6d7d52527518)
아까 **environment** 속성을 보면 `test`라고 설정해놓은 것을 볼 수 있습니다. 해당 환경이 여기에 적용됩니다.

### Branch rule 정의하기
이번에는 해당 Repository의 Settings -> Branches 탭으로 들어갑니다. 그리고 원하는 branch에 들어가 `edit` 버튼을 누릅니다.

그리고 사진과 같이 **Require deployments to succeed before merging** 속성을 클릭합니다. 그리고 아래와 같이 어떤 환경을 적용할 것인지 선택할 수 있습니다.

이 속성은 해당 Workflow의 Job이 성공해야 merge 할 수 있도록 브랜치를 보호하는 기능입니다.

그리고 저희는 frontend와 backend Job의 환경을 둘 다 test라는 이름으로 정의했기 때문에 하나의 environment만 선택해도 둘 다 적용되는 효과를 볼 수 있습니다.
![branch rule](https://github.com/car-ffeine/car-ffeine.github.io/assets/106640954/02be4679-56a2-4e47-ae01-b7025f8778a4)

#### 적용 후

아래와 같이 merge가 안된다는 글과 빨간색으로 경고 표시를 해주고 있습니다.
![blocked](https://github.com/car-ffeine/car-ffeine.github.io/assets/106640954/7dfba566-c8c8-4a24-a0e3-42081f3af31c)


## 결론

간단한 github action을 통해서 생산성을 많이 올릴 수 있는 좋은 기능인 것 같습니다. 다른 팀들도 이 기능을 도입하여 사용하는 것을 추천드립니다.


