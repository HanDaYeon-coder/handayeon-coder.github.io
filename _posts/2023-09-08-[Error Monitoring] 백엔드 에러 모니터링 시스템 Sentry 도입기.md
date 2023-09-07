---
title: "[Error Monitoring] 백엔드 에러 모니터링 시스템 Sentry 도입기"
date: 2023-09-08
categories: [Error Monitoring, Sentry]
---

# 0️⃣ 서론

저희 팀은 이전까지 에러 및 이슈와 관련된 문제들을 모니터링, 수집하기 위해서, 디스코드에 채널방을 만들어서 서로 어떠한 에러가 발생하고 있는지 작성해주고 있었습니다. 

![Alt text](/assets/img/2023-09-08/image-6.png)

![Alt text](/assets/img/2023-09-08/image-7.png)

그러나, 모든 에러를 디스코드에 공유하기는 어려웠고 실제 프로젝트를 사용자들에게 배포해서 운영할 때에는 에러를 따로 모니터링하기 힘들다고 판단하였습니다.

따라서 저희는 에러 모니터링 시스템을 도입하기로 하였고, 이중에서 무료로 사용할 수 있는 Sentry를 사용하기로 하였습니다.

Sentry의 계정을 만들고, Sentry에 프로젝트를 생성한 뒤에 백엔드 코드에 설정 파일를 작성하고 에러를 수집하고 싶은 곳에 에러 수집 코드만 작성하면 되어서 하루면 쉽게 에러 모니터링 시스템 Sentry를 도입할 수 있습니다!!

---

# 1️⃣ 본론 1

## Spring Boot에 Sentry 도입하기 🤖

저희 팀은 Spring Boot 2.7 을 사용하고 있습니다.
(해당 글은 Spring Boot 2 버전을 기준으로 작성되었습니다.)

아래의 sentry 공식 계정 가이드를 통해서 쉽게 설정하는 방법을 확인할 수 있습니다.

<a href="https://docs.sentry.io/platforms/java/guides/spring-boot/"> https://docs.sentry.io/platforms/java/guides/spring-boot/ </a>

### 1. Sentry에 로그인 한 후에 새로운 프로젝트 생성하기
- 이때, platform은 SPRING BOOT를 선택해줍니다.
- Alert frequency는 자신이 원하는 알림 주기로 선택해줘도 됩니다.
- 저희 팀은 아직 운영을 시작하지 않아서, 알림이 많이 오지 않을 것으로 예상되어서 모든 이슈에 대해서 알림을 하도록 설정하였습니다.
- 그러나, 무료 버전은 리포팅의 개수가 제한되어 있어서 알림 주기를 신중하게 선택해야 합니다.

![Alt text](/assets/img/2023-09-08/image.png)

### 2. Client Keys(DSN)의 DSN의 키 복사하기
- Sentry의 프로젝트를 만들 때에도 DSN 키가 바로 나옵니다. 이때, 복사하지 못했다면 Settings 메뉴에서도 확인할 수 있습니다.

![Alt text](/assets/img/2023-09-08/image-8.png)

### 3. build.gradle에 sentry 의존성 추가하기

`implementation 'io.sentry:sentry-spring-boot-starter:6.28.0'`

### 4. application.yml 파일에 DSN 설정하기
- sentry.dsn에 아까 위에서 복사한 dsn 키를 붙여넣습니다.

```yml
sentry:
  dsn: 복사한 DSN 키
  enable-tracing: true
```

- 이때, 각각의 환경(개발, 운영, 로컬)을 설정할 수 있습니다.
- 해당 파일은 application-dev.yml과 같이 각각의 환경 설정 파일에 추가해줘야 합니다.
```yml
sentry:
  dsn: 복사한 DSN 키
  enable-tracing: true
  environment: development
```
- 그러면, 아래의 사진과 같이 environment를 sentry 이슈에서 확인할 수 있습니다. (환경 별로 이슈를 분리할 수 있습니다.)

![Alt text](/assets/img/2023-09-08/image-10.png)

### 5. Sentry로 에러 전송하기

- 전송할 에러를 `Sentry.captureException(e);` 함수를 사용하여, Sentry에 에러를 전송할 수 있습니다.
- 이외에도 에러를 전송하는 함수가 많아서, 상황에 맞게 함수를 선택하면 됩니다.
- 현재 저희 프로젝트는 `@RestControllerAdvice`로 공통 예외 처리를 진행하고 있어서, 해당 ControllerExceptionAdvice 클래스에 에러 전송하는 코드를 추가하였습니다.

```java
@RequiredArgsConstructor
@RestControllerAdvice
public class ControllerExceptionAdvice {

    /**
     * 400 Bad Request (잘못된 요청, Validation Exception)
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(BindException.class)
    private ApiResponse<Object> handleValidationError(BindException e) {
        String errorMessage = e.getBindingResult().getFieldErrors().stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(joining("\n"));
        Sentry.captureException(e);
        return ApiResponse.error(ErrorCode.INVALID, errorMessage);
    }

    /**
     * 400 Bad Request (잘못된 요청)
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(InvalidException.class)
    private ApiResponse<Object> handleBadRequest(InvalidException e) {
        Sentry.captureException(e);
        return ApiResponse.error(e.getErrorCode());
    }

    /**
     * 404 Not Found (존재하지 않는 리소스)
     */
    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ExceptionHandler(NotFoundException.class)
    private ApiResponse<Object> handleNotFound(NotFoundException e) {
        Sentry.captureException(e);
        return ApiResponse.error(e.getErrorCode());
    }

    /**
     * 500 Internal Server Exception (서버 내부 에러)
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(InternalServerException.class)
    private ApiResponse<Object> handleInternalServerException(InternalServerException e) {
        Sentry.captureException(e);
        return ApiResponse.error(e.getErrorCode());
    }
}

```

---

# 2️⃣ 본론 2

## 실제로 Sentry 사용해보기 🍅
여기까지 하면 Sentry를 모두 다 설정하였습니다!! 

따라서, 실제로 에러를 발생시키고 Sentry로 에러 관련 정보가 잘 전송되었는지 확인해보겠습니다⁉️

### 1. BindException 예외(에러)를 발생시킨다.
- 아무것도 입력하지 않고 스크랩 추가하여 MethodArgumentNotValidException을 발생시키겠습니다.

![Alt text](/assets/img/2023-09-08/image-5.png)

![Alt text](/assets/img/2023-09-08/image-12.png)

2023-09-08 00:48:52.703  WARN 7838 --- [nio-8080-exec-4] .m.m.a.ExceptionHandlerExceptionResolver : Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public com.forever.dadamda.dto.ApiResponse<com.forever.dadamda.dto.scrap.CreateScrapResponse> com.forever.dadamda.controller.scrap.ScrapController.addScraps(com.forever.dadamda.dto.scrap.CreateScrapRequest,org.springframework.security.core.Authentication) throws net.minidev.json.parser.ParseException: [Field error in object 'createScrapRequest' on field 'pageUrl': rejected value []; codes [NotBlank.createScrapRequest.pageUrl,NotBlank.pageUrl,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [createScrapRequest.pageUrl,pageUrl]; arguments []; default message [pageUrl]]; default message [url을 입력해주세요.]] ]

![Alt text](/assets/img/2023-09-08/image-4.png)

- 위의 에러 메시지는 위에서 발생시킨 에러를 설명하기 위한 사진과 메시지이므로 아래의 Sentry에 표시된 시간과 다릅니다.

### 2. Sentry의 이슈들을 확인해줍니다.
- 아래의 사진에서 'MethodArgumentNotValidException'이 생성되었음을 알 수 있습니다.

![Alt text](/assets/img/2023-09-08/image-2.png)

- 해당 이슈를 클릭하면 어떤 브라우저, 디바이스 환경, 환경(개발, 운영, 로컬), 서버, url에서 발생한 에러인지  상세하게 확인할 수 있습니다.

![Alt text](/assets/img/2023-09-08/image-11.png)

---

# 3️⃣ 본론 3

## 🚨 주의할 점 

Sentry를 사용하면서 주의할 점에 대해서 이야기해보겠습니다!

### 1. 무료 계정으로 만들기

![Alt text](/assets/img/2023-09-08/image-3.png)

사진 출처 : https://sentry.io/pricing/

무료 개정은 Developer 계정입니다.

따라서, 제한된 모니터링 가능한 에러 개수, 1명만 사용 가능 등 여러 제한 사항이 있지만 무료로 사용할 수 있다는 가장 큰 메리트가 있어서 Developer 계정으로 진행하였습니다.

그러나, 회원가입하고 나서 가격을 선택하는 페이지가 나올 줄 알았는데😢 바로 Team 계정으로 생성되었고 Setting에서도 Developer 계정으로 변경하기는 어려웠습니다.

따라서 계정을 탈퇴하고 나서, 해당 페이지에서 <a href="https://sentry.io/pricing/"> https://sentry.io/pricing/ </a> Developer Pricing의 GET STARTED 버튼을 눌러서 계정을 새롭게 만들었습니다.

![Alt text](/assets/img/2023-09-08/image-13.png)

따라서 설정으로 이동하면 Developer Plan임을 확인할 수 있습니다.

다른 Team, Business 계정은 돈이 청구되는 만큼 현재 프로젝트에 알맞는 plan인지 확인해보는 것도 중요합니다!!

### 2. local은 기록하지 않기

현재 무료 계정을 사용하고 있기 때문에 사용 가능한 에러의 개수가 제한되어 있습니다.

local에서는 Sentry를 사용하지 않고 dev, prod와 같이 실제 에러를 수집해야 하는 환경에만 설정하는 것이 좋다고 판단하였습니다.

하지만, Sentry를 local의 환경 설정 파일(yml)에 설정해주지 않으면 에러가 발생합니다.

따라서 검색해서 찾아본 결과, 아래의 글에서 추천한 방법으로 진행하였습니다.

![Alt text](/assets/img/2023-09-08/image-14.png)

- dsn을 입력하는 자리에 아무것도 기록하지 않도록 하였습니다.

<a href="https://github.com/getsentry/sentry-symfony/issues/38"> https://github.com/getsentry/sentry-symfony/issues/38 </a>

### 2. 사용자의 실수도 기록해야 하는가?
- 현재 Validation Error를 수집하도록 하였습니다.
- 그러나, Validation Error는 개발자의 실수가 아닌 사용자의 실수이기 때문에 에러 처리만 잘 되어 있다면 수집할 정도로 중요한 에러가 아닙니다.
- 사용자의 실수를 수집한다면 에러가 계속 쌓여서 실제로 개발자가 수집해야 할 중요한 에러들을 관리하고 파악하는 것이 어렵습니다.
- 하지만, 사용자들이 많이 하는 실수들을 기록하고 더 나은 UIUX를 개선하는 등 사용자의 실수들을 기록하고 싶은 상황도 있을 수 있으므로 상황에 맞는 에러 관리가 필요하다고 생각했습니다!

```java
@RequiredArgsConstructor
@RestControllerAdvice
public class ControllerExceptionAdvice {

    /**
     * 400 Bad Request (잘못된 요청, Validation Exception)
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(BindException.class)
    private ApiResponse<Object> handleValidationError(BindException e) {
        String errorMessage = e.getBindingResult().getFieldErrors().stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(joining("\n"));
        Sentry.captureException(e);
        return ApiResponse.error(ErrorCode.INVALID, errorMessage);
    }
}

```

---

#  3️⃣ 결론 
![Alt text](/assets/img/2023-09-08/image-15.jpg)

- 이전의 Discord를 사용해서 에러를 관리할 때보다 Sentry를 도입하여 에러를 수집하고 관리하니까 훨씬 편리하다고 판단하였습니다.
- 현재 Slack이나 Jira 등 다른 의사소통 관리 도구와 연동하지 않아서 에러를 즉각적으로 파악하기가 어려워서, 다음에는 Slack, Jira에 연동해야겠다는 다짐을 하게 되었습니다.
- 에러는 개발자에게 매우 중요한 정보이기 때문에 에러를 모니터링하고 파악하는 것도 굉장히 개발자의 중요한 역량이라고 생각했습니다.
- 처음으로 에러 모니터링 시스템을 도입하고 에러가 발생한 브라우저, 디바이스 환경을 알려주는 모습을 통해서,이전까지는 어떠한 에러가 발생하였는지만 알면 되지 않을까라는 생각을 했는데 에러가 브라우저, 디바이스 환경마다 다를 수 있기 때문에 에러에서 파악해야 할 요소가 굉장히 많음을 깨달았습니다!
- 그리고 에러 모니터링 시스템이라고 해서 실제로 진행하고 있는 프로젝트 환경에 적용하기 어렵지 않을까 고민했는데, 그러한 고민이 무의미해질 정도로 굉장히 쉽고 빠르게 적용되는 모습을 보고 역시 뭐든지 두려워하지 않고 시도해봐야겠다는 다짐을 다시 한 번 더 하게 되었습니다.
