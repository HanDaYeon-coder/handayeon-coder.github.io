---
title: "[HTTP API URI 설계] 프로젝트에 컨트롤 URI 설계하기"
date: 2023-10-05
categories: [HTTP, API, URI]
---

# 0️⃣ 서론

이번에 프로젝트에서 보드 관련된 기능을 제공하기로 하였습니다.

이때, 저희는 보드 관련 기능을 아래와 같이 정의하였습니다.

1. 보드 생성
2. 보드 삭제
3. 보드 수정
4. 보드 조회
5. 보드 고정

이때, 다른 기능들은 CRUD로 정의가 되지만, 5번의 보드 고정은 CRUD로 정의할 수 없었습니다.

보드 고정은 아래의 피그마의 예시와 같이 스크랩들을 담은 보드를 고정핀(별)으로 고정하게 된다면, 보드 목록 내에서 맨 위에 고정되어 맨처음에 조회할 수 있는 기능을 만들고자 하였습니다.

![Alt text](/assets/img/2023-10-06/image.png)

저는 해당 기능의 HTTP API의 URI를 설계하려고 했습니다.

그러나, HTTP 메소드는 GET, POST, PUT, PATCH, DELETE로 이루어져 있어서 리소스와 행위를 분리하는 방식을 적용해서 보드를 고정하는 행위를 HTTP의 메소드로 나타내기가 어려웠습니다.

따라서 김영한님의 인프런 HTTP 강의를 듣고 나서, 프로젝트에 어떻게 적용시켜야할지를 되돌아보았습니다!

해당 강의 주소 : https://www.inflearn.com/course/http-웹-네트워크/dashboard

---

# 1️⃣ 본론

## 1. HTTP API URI 설계
### 여기에서 가장 중요한 점은 "리소스 식별"입니다.

즉, 위에서 이야기한 보드의 기능을 살펴보면서 이야기하겠습니다.

아래의 보드 기능에서 리소스는 바로 "보드"입니다.

1. 보드 생성 -> /boards
2. 보드 삭제 -> /boards/{id}
3. 보드 수정 -> /boards/{id}
4. 보드 조회 -> /boards/{id}
5. 보드 고정 -> /boards/{id}

### 리소스와 행위를 분리해야 합니다.
- URI는 리소스만 식별합니다.
- 리소스와 해당 리소스를 대상으로 하는 행위를 분리합니다.
- 행위는 "생성, 삭제, 수정, 조회, 고정"입니다.

### 행위는 HTTP 메소드로 구분합니다.
주요 메소드
1. GET : 리소스를 조회합니다.
2. POST : 요청 데이터 처리, 주로 등록에 사용됩니다.
3. PUT : 리소스 대체, 해당 리소스가 없으면 생성합니다.
4. PATCH : 리소스 부분을 변경합니다.
5. DELETE : 리소스를 삭제합니다.

---

### Control URI (컨트롤 URI)
- 단순한 데이터를 생성하거나, 변경하는 것을 넘어서 프로세스를 처리해야 하는 경우에 사용됩니다.
- 예를 들어서, 주문에서 결제완료 -> 배달 시작 -> 배달 완료처럼 단순히 값 변경을 넘어 프로세스의 상태가 변경되는 경우에 사용될 수 있습니다.
  - POST `/orders/{orderId}/start-delivery`

## 2. 프로젝트에 적용하기

저희 프로젝트는 API의 버전 관리를 위해서 v1을 맨 처음에 붙이기로 하였습니다.

그리고 고정하다를 "fix"로 하여 API를 설계해줍니다.

따라서 `/v1/boards/{boardId}/fix`로 해주었습니다. 

그러나, 과연 boardId가 가운데에 API URI로 들어가는 것이 좋을까??라는 의문이 들었습니다.

가운데에 넣으면 URI의 Validation 유효성 검사를 하기가 어려워질 것 같다는 생각을 했습니다.

`/v1/boards/{boardId}/fix`를 한다면, boardId가 null일 때에는 boardId가 fix로 인식되어서 @NotNull을 못 잡아낼 수도 있을 것 같았습니다.

따라서 {boardId}를 마지막에 한다면, /v1/boards/fix와 /v1/boards/fix/1 처럼 null과 boardId 여부가 명확해져서 null 관련 validation 처리가 좋을 것 같다고 판단하였습니다.

`/v1/boards/{boardId}/fix`로 작성해도 유효성 검사가 잘 작동될지를 검토하고, URI를 선택해보겠습니다.

service 로직을 검증하는 것이 아니므로 간단하게 controller 테스트를 위해서 간단하게 아래의 코드로 시험해보겠습니다!

### 1. /v1/boards/{boardId}/fix

이때의 API URI만 테스트하기 위해서, POST와 PATCH로 나눠서 진행하였습니다.

따라서 해당 테스트에서는 HTTP 메소드에 크게 의미를 부여하지 않고 있습니다.

```java
@Operation(summary = "보드 고정", description = "1개의 보드를 보드 카테고리에서 상단에 고정합니다.")
@PostMapping("/v1/boards/{boardId}/fix")
public ApiResponse<String> fixBoards(@PathVariable(required = false) @Positive @NotNull Long boardId,
        Authentication authentication) {
    String email = authentication.getName();
    return ApiResponse.success("boardId : " + boardId + " email : " + email);
}
```

![Alt text](/assets/img/2023-10-06/image-1.png)

![Alt text](/assets/img/2023-10-06/image-3.png)

### 2. /v1/boards/fix/{boardId}

```java
@Operation(summary = "보드 고정", description = "1개의 보드를 보드 카테고리에서 상단에 고정합니다.")
@PatchMapping("/v1/boards/fix/{boardId}")
public ApiResponse<String> fixedBoards(@PathVariable(required = false) @Positive @NotNull Long boardId,
        Authentication authentication) {
    String email = authentication.getName();
    boardService.fixedBoards(email, boardId);
    return ApiResponse.success();
}
```

![Alt text](/assets/img/2023-10-06/image-5.png)

![Alt text](/assets/img/2023-10-06/image-6.png)

이때, 두 가지의 URI 모두 `@NotNull`임을 확인하지 못하고 에러가 난다는 사실을 알 수 있습니다.

따라서 Controller에서는 @NotNull인 상황은 기본적으로 처리해주지 못하기 때문에 `/v1/boards/{boardId}/fix`와 `/v1/boards/fix/{boardId}`가 동일한 null 처리를 추가해주어야 하기 때문에 큰 차이가 없을 것이라고 생각했습니다.

### 참고 자료
- https://memo-the-day.tistory.com/108
- https://www.baeldung.com/spring-optional-path-variables

---

## 3. @NotNull 처리

위의 두 개를 @NotNull인 경우를 처리를 해보겠습니다.

```java
@Operation(summary = "보드 고정", description = "1개의 보드를 보드 카테고리에서 상단에 고정합니다.")
@PatchMapping(value = {"/v1/boards/fix/{boardId}", "/v1/boards/fix"})
public ApiResponse<String> fixedBoards(@PathVariable(required = false) @Positive @NotNull Long boardId,
        Authentication authentication) {
    String email = authentication.getName();
    boardService.fixedBoards(email, boardId);
    return ApiResponse.success();
}
```
![Alt text](/assets/img/2023-10-06/image-2.png)

```java
@Operation(summary = "보드 고정", description = "1개의 보드를 보드 카테고리에서 상단에 고정합니다.")
@PostMapping(value = {"/v1/boards/{boardId}/fix", "/v1/boards/fix"})
public ApiResponse<String> fixBoards(@PathVariable(required = false) @Positive @NotNull Long boardId,
        Authentication authentication) {
    String email = authentication.getName();
    return ApiResponse.success("boardId : " + boardId + " email : " + email);
}
```
![Alt text](/assets/img/2023-10-06/image-4.png)

따라서, NotNull 처리가 가능합니다.

많은 예시를 찾아보았을 때, NotNull을 처리하는 경우에는 대부분 null로 들어오면 디폴트 값을 설정해주어서 에러가 나지 않게 하는 방향으로 많이 하는 방향으로 많이 했습니다.

저희 서비스는 보드 ID가 사용자마다 다르기 때문에, 임의 값을 지정해줄 수 없어서 디폴트 값 처리가 필요하지 않습니다.

그러나, 아래와 같이 @NotNull인 경우에는 404 NOT FOUND 에러가 아닌 500 Internal Server Error가 나오기 때문에, Null로 입력한 클라이언트 오류가 서버 오류가 되어서 최대한으로 500에러를 안내는 것이 서버 API가 좋다고 판단하여서 아래의 사진 밑 코드로 프로젝트에 반영하기로 하였습니다!

![Alt text](/assets/img/2023-10-06/image-5.png)

```java
@Operation(summary = "보드 고정", description = "1개의 보드를 보드 카테고리에서 상단에 고정합니다.")
@PostMapping(value = {"/v1/boards/{boardId}/fix", "/v1/boards/fix"})
public ApiResponse<String> fixBoards(@PathVariable(required = false) @Positive @NotNull Long boardId,
        Authentication authentication) {
    String email = authentication.getName();
    return ApiResponse.success("boardId : " + boardId + " email : " + email);
}
```

---

# 2️⃣ 결론

- HTTP API URI 설계가 쉬운 듯 해보이지만, 클라이언트와의 약속 및 API 서버로 들어오기 위한 출입구 같은 역할을 하기 때문에 잘 설계하는 것이 매우 중요하고, 팀끼리 정하기에 따라서 달리지는 만큼 사람들마다 다르게 설계하여 명확한 기준이 없어서 어려운 부분도 있었습니다.
- 그러나, 이번 기회에 HTTP API를 다시 한 번 살펴보고 처리를 어떻게 해야할지 테스트를 해보면서 더 많이 알게 된 것 같았습니다.
- 그리고 이전에는 GET, POST, PATCH, DELETE와 같이 CRUD 기본적인 것들만 API를 구현하다보니까, 컨트롤 URI를 처음으로 사용하게 되었는데, 실제 서비스에서는 이것보다 더 훨씬 복잡한 로직을 수행하는 API를 구현할 수도 있는 만큼 이번에 했던 것들을 떠올리면서 더 좋은 설계를 위해서 힘써야겠다는 다짐을 하게 되었습니다.