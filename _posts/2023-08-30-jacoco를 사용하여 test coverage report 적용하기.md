---
title: "jacoco를 사용하여 test coverage report 적용하기"
date: 2023-08-30
categories: [Jacoco, Test Coverage Report]
---

# 0️⃣ 서론

이번에 프로젝트에 테스트 코드를 도입하면서, 내가 작성한 테스트 코드의 커버리지 결과가 얼마나 나오는지 궁금했습니다! 

이러한 커버리지 결과를 인텔리제이에서는 바로 확인이 가능하지만, Github에 PR로 올리면 다른 팀원들이 코드 커버리지를 확인할 수 없습니다.

따라서 이러한 PR을 올릴 때 코드 커버리지를 알려주는 라이브러리를 도입해서 팀원들과 같이 커버리지를 공유하려고 했습니다. 

그러나, 관련 라이브러리를 사용하면서 Github의 PR에 자동으로 올리는 작업이 생각보다 오류가 많이 발생했고, 이를 해결하기 위해서 여러 시도를 했고 마침내 해결했습니다! 

이러한 삽질 과정을 같이 공유해서 다른 분들은 이러한 과정을 최소화했으면,,, 좋겠다는 바램으로 글을 작성합니다!

<br/>

# 1️⃣ 본론

## PART 1. Jacoco가 그래서 뭘까?? 🌴

먼저, `Jacoco`를 사용하기 전에, 도대체 Jacoco가 무엇인지 왜 사용해야 하는 것인지 알아보도록 하겠습니다!

Jacoco는 `자바 코드 커버리지`를 확인하는 데에 사용되는 오픈소스 라이브러리입니다.

테스트 코드를 돌리고 나서, 아래의 그림과 같이 자바 코드 커버리지의 결과를 사람들이 쉽게 파악할 수 있도록 xml, html 형태의 리포트를 생성해줍니다.

또한 설정한 커버리지 기준을 만족하지 않는 것이 있는지도 확인해줍니다.

![Alt text](/assets/img/2023-08-30/image-23.png)

출처 : <a href="https://www.jacoco.org/jacoco/"> https://www.jacoco.org/jacoco/ </a>

따라서, Jacoco를 사용하여 코드 커버리지 결과를 xml 형태의 리포트로 만들고 나서 해당 xml 파일을 PR의 커멘트로 남기도록 하겠습니다!!

아래의 사진은 Jacoco로 생성된 xml 파일을 PR 커멘트로 남긴 예시입니다~

![Alt text](image.png)

<br/>

## PART 1. 프로젝트에 JaCoCo 적용하기 🥥

### 1. build.gradle 설정하기 (JaCoCo 플러그인 추가)

```gradle
plugins {
    id 'jacoco'
}

test {
    finalizedBy jacocoTestReport
    useJUnitPlatform()
}

jacoco {
    toolVersion = "0.8.8"
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.enabled true
    }
}

```

위의 설정 파일을 자세하게 살펴보면,

```gradle
test {
    finalizedBy jacocoTestReport
    useJUnitPlatform()
}
```

위의 문구는 test 작업을 수행하고 나서, jacocoTestReport 작업을 수행하라는 의미입니다.

```gradle
jacocoTestReport {
    dependsOn test
    reports {
        xml.enabled true
    }
}
```

jacocoTestReport를 생성할 때 xml 파일만 생성하도록 설정하였습니다.

html 파일도  html.enabled true 문장을 추가로 선언하면 같이 생성되도록 할 수 있습니다.

이때, Github Actions의 PR 커멘트를 추가할 때는 xml 파일을 사용할 예정이기 때문에 반드시 xml.enabled true를 하여 xml 파일이 생성되도록해야 합니다.

설정 파일에서 jacocoTestReport의 경로를 따로 명시하지 않을 경우, 아래 사진과 같이
build/reports/jacoco/test/jacocoTestReport.xml 경로에 파일이 생성됩니다.


![Alt text](/assets/img/2023-08-30/image-1.png)


### 2. Jacoco-report 사용해서, 커버리지 리포트를 PR 코멘트로 생성하기

>> A Github action that publishes the JaCoCo report as a comment in the Pull Request with customizable pass percentage for modified modules, files and the overall project. You can view the coverage of just the changed files in your pull request.

출처 : https://github.com/Madrapps/jacoco-report

Jacoco-report는 Jacoco에 의해 build 시점에 생성되는 리포트 xml파일을 이용해
PR에 코멘트를 생성해주는 Github Action 라이브러리입니다.

```yml
name: PR Test

on:
  pull_request:
    branches: [ develop ]

permissions: write-all

jobs:
  test:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'corretto'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Test with Gradle
        run: ./gradlew --info test

      - name: Test Coverage Report
        id: jacoco
        uses: madrapps/jacoco-report@v1.6
        with:
          title: Test Coverage Report
          paths: ${{ github.workspace }}/build/reports/jacoco/test/jacocoTestReport.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          min-coverage-overall: 30
          min-coverage-changed-files: 50
```

이때, 유의할 점은 
`${{ github.workspace }}`, `${{ secrets.GITHUB_TOKEN }}`은 
Github action의 기본 설정이기 때문에 github secrets 파일에 따로 직접 설정해줄 필요가 없습니다!

paths 관련된 설정은 위에서 build.gradle 설정하기 (JaCoCo 플러그인 추가)에서 따로 
jacocoTestReport 경로를 지정하지 않았다면 그대로 `paths: ${{ github.workspace }}/build/reports/jacoco/test/jacocoTestReport.xml`를 사용하면 됩니다.

- min-coverage-overall는 프로젝트 전체 테스트 커버리지에 대한 최소 코드 커버리지 기준입니다.
- min-coverage-changed-files는 변경된 파일에 대한 최소 코드 커버리지 기준입니다.
- 따로 설정하지 않으면, 두 개 모두 default의 값은 80%입니다.

그 외의 설정할 수 있는 것들은 아래의 사진을 통해서 살펴볼 수 있습니다.

![Alt text](/assets/img/2023-08-30/image-2.png)

`uses: madrapps/jacoco-report@v1.6` 

여기에서 v1.6은 시간이 변경될 때마다 업데이트되므로 아래의 링크를 통해서 tags를 살펴보고 이중에서 원하는 버전으로 변경해도 됩니다.

<a href="https://github.com/Madrapps/jacoco-report"> https://github.com/Madrapps/jacoco-report </a>

![Alt text](/assets/img/2023-08-30/image-3.png)

참고 자료 : <a href="https://creampuffy.tistory.com/193"> https://creampuffy.tistory.com/193 </a>

<br/>

## PART 3. 설정 과정에서 발생한 문제점과 해결책 🧚‍♀️

### 1. Can't add different class with same name

local의 인텔리제이에서 `./gradlew test` 했을 때, 다음과 같이 jacocoTestReport를 돌릴 수 없다고 나왔다.
그래서 왜인지 자세하게 살펴보기 위해서 jacocoTestReport 결과를 살펴보았는데,
동일한 이름의 다른 클래스를 추가할 수 없다고 나왔습니다.

![Alt text](/assets/img/2023-08-30/image-4.png)

![Alt text](/assets/img/2023-08-30/image-5.png)

그런데 동일한 이름의 클래스를 만든 적이 없었고, 이름을 변경해주었더니 다른 파일들도 동일한 오류 메시지가 뜨게 되었습니다.

도대체 왜 그럴까?? 계속 찾아보니까, ./gradlew test을 하고 나서 또 다시 동일한 명령어를 쳤을 때 발생하는 에러였습니다. 처음 ./gradlew test를 작동시켰을 때에는 발생하지 않는 에러였습니다. 

따라서 build해서 생성된 파일에서 에러가 발생한 파일들이 이미 존재하고, 또 한번  ./gradlew test를 하면 build해서 생성된 파일들과 동일한 이름의 파일을 만들려고 해서 에러가 발생하였습니다.

그래서 `./gradlew clean test`을 해서 build해서 생성된 파일을 지우고 test를 실행하면 에러가 발생하지 않았습니다!

![Alt text](/assets/img/2023-08-30/image-6.png)


### 2. test coverage report 작성시 발생한 문제점


```yml
name: PR Test

on:
  pull_request:
    branches: [ develop ]

permissions: write-all

jobs:
  test:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'corretto'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Test with Gradle
        run: ./gradlew --info test

      - name: Test Coverage Report
        id: jacoco
        uses: madrapps/jacoco-report@v1.6
        with:
          title: Test Coverage Report
          paths: ${{ github.workspace }}/build/reports/jacoco/test/jacocoTestReport.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          min-coverage-overall: 30
          min-coverage-changed-files: 50
```

Github actions에서 Test Coverage Report 부분에서 에러가 발생하였습니다.

`Error: HttpError: Resource not accessible by integration` 라는 에러였습니다.

![Alt text](/assets/img/2023-08-30/image-7.png)

열심히 찾아보니까, jacoco-report 라이브러리에서 리포트를 PR 커멘트로 작성해야 하는데 Pull request에 작성할 권한이 없다는 문제였습니다.

![Alt text](/assets/img/2023-08-30/image-8.png)

`permissions: write-all`을 붙여주었는데,,,, 왜 도대체 왜,,, 계속 read로만 나올까??라는 의문을 가지고 계속 여러번 시도했습니다.

해당 이슈는 https://github.com/Madrapps/jacoco-report 라이브러리에서도 소개할 정도로 문제가 많이 발생하는 것 같습니다... 🥹

- https://github.com/Madrapps/jacoco-report/issues/24

![Alt text](/assets/img/2023-08-30/image-10.png)

권한을 주기 위해서 Workflow permissions 권한을 Read and Write permissions, Allow GitHub actions to create and approve pull requests도 허용했습니다.

- 변경 전
![Alt text](/assets/img/2023-08-30/image-9.png)
- 변경 후
![Alt text](/assets/img/2023-08-30/image-12.png)

- 이때, Read and Write permissions, Allow GitHub actions to create and approve pull requests이 눌리지 않는다면, 원하는 repository의 Organization의 Settings로 이동하고, actions > general로 들어가면 동일한 workflow permissions가 있고 여기에서는 Read and Write permissions, Allow GitHub actions to create and approve pull requests을 선택할 수 있습니다.
![Alt text](/assets/img/2023-08-30/image-13.png)
![Alt text](/assets/img/2023-08-30/image-14.png)

이렇게 설정해줘도, Pull request 권한은 read였습니다.

![Alt text](/assets/img/2023-08-30/image-8.png)

GitHub Docs에서 GITHUB_TOKEN의 권한에 대한 설명을 읽어보면,
`Maximum access for
pull requests from
public forked repositories`에서 모두다 read가 됨을 알 수 있습니다.

![Alt text](/assets/img/2023-08-30/image-15.png)

출처 : https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token


🚨 현재 저희 프로젝트에서 진행하고 있는 Git-Flow 전략을 살펴보겠습니다.

아래의 우아한 형제들의 Git-Flow 전략과 같이 각자의 Local Repository에서 Fork한 Origin romote repository로 push하고 Upstream Romote Repository로 Pull Request를 보내는 구조입니다.

![Alt text](/assets/img/2023-08-30/image-17.png)
그림 출처 : https://techblog.woowahan.com/2553/

지금 살펴보고 있는 Github-Actions는 HanDaYeon-coder:test/DAD-710(Fork한 Origin romote repository)에서 SWM-team-forever:develop(Upstream Romote Repository)로 Pull Request를 보낸 뒤에 실행되는 구조입니다.

이 구조는 `Maximum access for
pull requests from
public forked repositories`이기 때문에 GITHUB_TOKEN의 권한은 모두 read가 되고 변경할 수 없습니다!

![Alt text](/assets/img/2023-08-30/image-16.png)

현재 구조로는 쓰기 권한이 없기 때문에 test coverage report를 Pull Request에 코멘트로 작성할 수 없습니다. 

따라서 현재 구조를 변경해서, 적용해보겠습니다!

1. HanDaYeon-coder:test/DAD-710(Fork한 Origin romote repository)에서 SWM-team-forever:test/DAD-710(Upstream Romote Repository)로 Pull Request를 보낸 뒤에 머지한다.

![Alt text](/assets/img/2023-08-30/image-18.png)

2. SWM-team-forever:test/DAD-710에서 SWM-team-forever:develop으로 Pull Request를 보내는 구조로 변경하였습니다.

![Alt text](/assets/img/2023-08-30/image-19.png)

그러면 GITHUB_TOKEN의 권한이 write로 변경되는 모습을 확인할 수 있습니다!

![Alt text](/assets/img/2023-08-30/image-21.png)

![Alt text](/assets/img/2023-08-30/image-22.png)

정상적으로 Test Coverage Report가 Pull Request 커멘트로 등록되는 모습을 확인할 수 있습니다~!
![Alt text](image.png)

<br/>

# 3️⃣ 결론

Test Coverage Report를 자동으로 Github에 보여주기 위해서, 굉장히 많은 시도와 검색을 하게 되었습니다.

지금까지는 많은 시간이 걸렸지만, 앞으로 자동화되면서 더 투자한 시간보다 많은 시간을 절약해줄 거라고 믿으면서 계속해서 시도를 했던 것 같습니다... 😅

그리고 Test Coverage Report 관련 블로그글을 보았을 때는 설정할 게 많이 없어서, 오랜 시간이 걸리지 않을 줄 알고 시도하긴 했지만 생각보다 많은 오류 때문에 포기하고도 싶었지만 마침내 PR 커멘트의 사과🍏를 보는 순간 모든게 잊혀진 순간이었습니다~