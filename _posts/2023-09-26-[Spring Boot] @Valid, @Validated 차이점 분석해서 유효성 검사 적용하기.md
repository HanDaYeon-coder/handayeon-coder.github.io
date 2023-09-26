---
title: "[Spring Boot] @Valid, @Validated 차이점 분석해서 유효성 검사 적용하기"
date: 2023-09-26
categories: [Spring Boot, Validation]
---
# 0️⃣ 서론

이전에 클라이언트의 요청 관련 Validation을 @Valid를 사용해서 @NotBlank, @URL, @Size 어노테이션을 사용하여 DTO에서는 유효성 검사를 했습니다.

아래의 예시는 스크랩 생성할 때, url을 입력하지 않고 "" 공백을 입력하는 경우에 에러가 발생하고 클라이언트에게 에러 메시지를 보내주게 됩니다.

이러한 유효성 검사가 있어야만 사용자의 잘못된 입력으로 인한 서버 오류(sql에 null이 들어가면 안되는 경우 등)를 미리 처리하여 시간 및 리소스 낭비 등을 방지할 수 있습니다!

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class CreateScrapRequest {

    @NotBlank(message = "url을 입력해주세요.")
    @URL(message = "URL 형식이 유효하지 않습니다.")
    @Size(max = 2083, message = "최대 2083자까지 입력할 수 있습니다.")
    private String pageUrl;
}
```
따라서 아래와 같이 RestControllerAdvice인 ControllerExceptionAdvice의 handleValidationError로 잡아서 에러를 클라이언트에게 알려주었습니다.

```java
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
```

아래와 같이 resultCode, message 형태로 response를 반환함을 알 수 있습니다.

![Alt text](/assets/img/2023-09-26/image.png)

그러면 DTO를 사용하지 않고 **Parameter와 PathVariable로 입력받는 경우에 유효성 검사**를 위해서 @Valid를 사용했는데 이 과정에서 발생한 오류와 해결과정을 소개해드리겠습니다!

---

# 1️⃣ 본론

## 1. @Validated 어노테이션, @NotBlank 사용하기 
저는 처음에는 @RequestParam("keyword") 뒤에 @NotBlank를 붙여주고, Controller Class에 @Validated 어노테이션을 작성함으로써 유효성 검사를 하려고 했습니다.

```java
@Validated
@RequiredArgsConstructor
@RestController
public class ArticleController {
    @Operation(summary = "아티클 스크랩 검색", description = "스크랩을 검색할 수 있습니다.")
    @GetMapping("/v1/scraps/article/search")
    public ApiResponse<Slice<GetScrapResponse>> searchArticles(
            @RequestParam("keyword") @NotBlank String keyword,
            Pageable pageable,
            Authentication authentication) {

        if(keyword.isBlank()) {
            throw new InvalidException("검색어를 입력해주세요.");
        }
        String email = authentication.getName();

        return ApiResponse.success(articleService.searchArticles(email, keyword, pageable));
    }
}
```

그러나, 위의 코드에서 ControllerExceptionAdvice의 handleValidationError로 에러를 처리하지 못하고 원하는 응답값을 얻을 수 없었습니다.

아래의 사진처럼 Internal Server Error가 나왔습니다.

![Alt text](/assets/img/2023-09-26/image-1.png)

원하는 형태의 에러 응답 형식이 나오지 않았습니다.

따라서 아래와 같이 if 문을 사용해서 예외 처리하는 로직을 추가했습니다.

그러면, 예외 처리 로직이 추가될 때마다 if문이 늘어나기 때문에 DTO의 @NotBlank와 같은 어노테이션이 없는지 처리를 더 간결하게 할 수 없는지 고민해보았습니다.

```java
@Operation(summary = "스크랩 검색", description = "스크랩을 검색할 수 있습니다.")
@GetMapping("/v1/scraps/search")
public ApiResponse<Slice<GetScrapResponse>> searchScraps(
        @RequestParam("keyword") String keyword,
        Pageable pageable,
        Authentication authentication) {

    if(keyword.isBlank()) {
        throw new InvalidException("검색어를 입력해주세요.");
    }
    String email = authentication.getName();

    return ApiResponse.success(scrapService.searchScraps(email, keyword, pageable));
}
```

그러면 어떻게 에러를 처리해야 할까?? 유효성 검사니까 동일한 에러가 나오지 않을까?라는 의문을 가지다가 에러의 trace 응답값에 주목하게 되었습니다.

![Alt text](/assets/img/2023-09-26/image-2.png)

이전에는 유효성 검사 handler는 @ExceptionHandler(BindException.class)였습니다.

그러나, Parameter의 유효성 검사에서 발생한 예외는 BindException이 아닌 **ConstraintViolationException**였습니다.

따라서 아래와 같이 ConstraintViolationException 예외를 처리해주는 handler를 추가해주었습니다.

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
@ExceptionHandler(ConstraintViolationException.class)
private ApiResponse<Object> handleValidationParameterError(ConstraintViolationException e) {
    String errorMessage = e.getMessage();
    Sentry.captureException(e);
    return ApiResponse.error(ErrorCode.INVALID, errorMessage);
}
```

아래와 같이 원하는 형태의 에러 응답 형식을 반환할 수 있게 되었습니다.

![Alt text](/assets/img/2023-09-26/image-3.png)

---

## 2. @Valid와 @Validated 차이점

그러면 왜 비슷한 어노테이션에서 동일한 유효검사를 해주었는데 왜 다른 예외가 발생하고 어느 차이가 있을지 학습하고 싶었습니다.

### 2-1. @Valid 어노테이션 동작 원리

@Valid 어노테이션은 SR-303 표준 스펙(자바 진영 스펙)으로써 빈 검증기(Bean Validator)를 이용해 객체의 제약 조건을 검증하도록 지시하는 어노테이션입니다.

Controller 로 요청이 들어오면, Dispatcher Servlet을 걸쳐서 Controller가 호출이 됩니다.
이 과정에서 Dispatcher Servlet 에서 @Valid 를 찾아서 검증을 진행하게 됩니다.

@Valid를 통해 검증을 한 후, 만약 요구사항을 충족하지 못하여 실패한 경우 위의 두 예외를 발생시키게 됩니다.

@ModelAttribute 어노테이션으로 받은 파라미터에서 검증 오류가 발생한 경우 **BindException**이 발생하고
@RequestBody 어노테이션으로 받은 파라미터에서 검증 오류가 발생한 경우 **MethodArgumentNotValidException**를 발생시킵니다.

### 2-2. @Validated 어노테이션 동작 원리

@Validated는 **AOP 기반으로 메소드 요청을 인터셉터하여 처리**됩니다. 

@Validated를 클래스 레벨에 선언하면 해당 클래스에 유효성 검증을 위한 AOP의 어드바이스 또는 인터셉터(MethodValidationInterceptor)가 등록됩니다. 

그리고 해당 클래스의 메소드들이 호출될 때 AOP의 포인트 컷으로써 요청을 가로채서 유효성 검증을 진행합니다.

- 즉, validated 어노테이션은 createMethodValidationAdvice() 메서드를 통해 AOP Advice인 MethodValidationInterceptor를 등록합니다.

- 여기서 등록된 MethodValidationInterceptor가 메서드 요청을 가로채 유효성 검사를 진행하게 됩니다.

![Alt text](/assets/img/2023-09-26/image-5.png)

![Alt text](/assets/img/2023-09-26/image-4.png)

이러한 이유로 @Valid에 의한 예외는 MethodArgumentNotValidException이며, @Validated에 의한 예외는  ConstraintViolationException입니다.

참고 자료 1 : https://mangkyu.tistory.com/174

참고 자료 2 : https://kapentaz.github.io/spring/Spring-Boo-Bean-Validation-제대로-알고-쓰자/#

참고 자료 3 : https://wildeveloperetrain.tistory.com/158

참고 자료 4 : https://kdhyo98.tistory.com/80

---

# 3️⃣ 결론
- 스프링의 어노테이션을 사용하면서, 어노테이션의 동작하는 원리에 대해 더 깊이 있게 이해할 수 있는 시간이 된 것 같습니다.
- 이름과 기능이 비슷한 valid과 validated 어노테이션을 사람들의 코드를 참고하여 무조건 사용하기보다는 동작 원리를 이해하면서 나의 상황에 맞는 처리 방법을 알게 되었습니다.
- 라이브러리의 로직 및 스프링의 동작 원리(AOP, Dispatcher Servlet)을 알고 이해하며, 라이브러리가 처리하는 방식을 통해서 예외 처리 방법을 알게 된 것 같습니다.