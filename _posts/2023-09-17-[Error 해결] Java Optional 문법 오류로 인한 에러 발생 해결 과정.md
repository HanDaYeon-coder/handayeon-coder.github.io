---
title: "[Error] Java Optional 문법 오류로 인한 에러 발생 해결 과정"
date: 2023-09-17
categories: [Error, Java, Optional]
---
# 0️⃣ 서론

이전 글에서 작성한 WebClient 관련 리팩토링을 진행한 뒤에 develop 브랜치로 머지하고 dev 환경에 배포를 마쳤습니다.

https://velog.io/@da_na/WebClient-WebClient-사용해서-외부-API-호출하기

![Alt text](/assets/img/2023-09-17/image-2.png)

그리고 dev 환경에서 실제로 스크랩을 추가한 뒤에 스크랩이 잘 조회되는지 확인해보았습니다.

그러나,,, 아래와 같이 1개의 스크랩을 추가했는데, 아래의 사진에서 빨간색 박스로 되어 있는 것처럼 정상적인 스크랩 1개와 비정상적인 스크랩(아무것도 나오지 않는 스크랩)이 2개가 추가되었습니다.

![Alt text](/assets/img/2023-09-17/image-1.png)

그래서, dev DB를 확인해봤는데 2개의 스크랩이 추가되었습니다.

![Alt text](/assets/img/2023-09-17/image.png)

- Article 스크랩과 Other 스크랩 2개가 생성되었습니다.


그래서 WebClient 리팩토링 코드에서 어느 부분이 틀렸는지 앞으로 이러한 상황을 대비하기 위해서는 어떻게 해야할지를 이야기해보겠습니다!!

---

# 1️⃣ 본론 1

## 이전 코드

- 이전에는 WebClient의 에러가 발생하는 경우 NotFoundException처리가 되어 있기 때문에 따로 스크랩을 저장하지 않는 방향이었습니다.

```java
@Transactional
public JSONObject crawlingItem(String pageUrl) throws ParseException {
    Map<String, Object> bodyMap = new HashMap<>();
    bodyMap.put("url", pageUrl);
    WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();

    Map<String, Object> response = webClient.post()
            .bodyValue(bodyMap)
            .retrieve()
            .bodyToMono(Map.class)
            .block();

    JSONParser jsonParser = new JSONParser();

    Object obj = jsonParser.parse(response.get("body").toString());

    JSONObject jsonObject = (JSONObject) obj;

    return jsonObject;
}
```

```java
@Transactional
public Scrap saveScraps(User user, String pageUrl) throws ParseException {
    JSONObject crawlingResponse = webClientService.crawlingItem(pageUrl);

    String type = "";
    try {
        type = crawlingResponse.get("type").toString();
    } catch (NullPointerException e) {
        throw new NotFoundException(ErrorCode.NOT_EXISTS);
    }

    switch (type) {
        case "video":
            return videoService.saveVideo(crawlingResponse, user, pageUrl);
        case "article":
            return articleService.saveArticle(crawlingResponse, user, pageUrl);
        case "product":
            return productService.saveProduct(crawlingResponse, user, pageUrl);
    }
    return otherService.saveOther(crawlingResponse, user, pageUrl);
}
```

## 수정된 코드

- WebClient를 리팩토링하면서, WebClient 에러가 발생하면 null을 반환하여 other 스크랩으로 저장되도록 로직을 변경했습니다.

```java
@Transactional
public WebClientBodyResponse crawlingItem(String crawlingApiEndPoint, String pageUrl) {
    Map<String, Object> bodyMap = new HashMap<>();
    bodyMap.put("url", pageUrl);

    WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();

    try {
        WebClientResponse webClientResponse = webClient.post()
                .bodyValue(bodyMap)
                .retrieve()
                .onStatus(HttpStatus::is4xxClientError, clientResponse -> {
                    throw new RuntimeException("4xx");
                })
                .onStatus(HttpStatus::is4xxClientError, clientResponse -> {
                    throw new RuntimeException("5xx");
                })
                .bodyToMono(WebClientResponse.class)
                .block();

        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        return objectMapper.readValue(
                webClientResponse != null ? webClientResponse.getBody() : null,
                WebClientBodyResponse.class);


    } catch (Exception e) {
        return null;
    }
}
```

```java
@Transactional
public Scrap saveScraps(User user, String pageUrl) throws ParseException {
    WebClientBodyResponse crawlingResponse = webClientService.crawlingItem(crawlingApiEndPoint, pageUrl);

    return Optional.ofNullable(crawlingResponse)
            .map(response -> {
                String type = response.getType();
                switch (type) {
                    case "video":
                        return videoService.saveVideo(response, user, pageUrl);
                    case "article":
                        return articleService.saveArticle(response, user, pageUrl);
                    case "product":
                        return productService.saveProduct(response, user, pageUrl);
                    case "place":
                        return placeService.savePlace(response, user, pageUrl);
                    default:
                        return otherService.saveOther(response, user, pageUrl);
                }
            })
            .orElse(otherService.saveOther(new WebClientBodyResponse(), user, pageUrl));
}
```

- 여기에서 잘못된 부분은 `.orElse(otherService.saveOther(new WebClientBodyResponse(), user, pageUrl));` 이었습니다.
- orElse 문을 사용하면 안되는 이유를 아래의 문법 관련해서 같이 설명드리겠습니다.

## 📄 Optional orElse문

**Java Optional 클래스**는 Java 8에서 추가되었으며 NullpointerException 문제를 해결할 수 있는 방법을 제공합니다.

Optional<T> 클래스는 Integer나 Double 클래스처럼 'T'타입의 객체를 포장해 주는 래퍼 클래스(Wrapper class)입니다.
따라서 Optional 인스턴스는 모든 타입의 참조 변수를 저장할 수 있습니다.
 
이러한 Optional 객체를 사용하면 예상치 못한 NullPointerException 예외를 제공되는 메소드로 간단히 회피할 수 있습니다.

즉, 복잡한 조건문 없이도 널(null) 값으로 인해 발생하는 예외를 처리할 수 있게 됩니다.

### Optional.ofNullbale() - 값이 Null일수도, 아닐수도 있는 경우

만약 어떤 데이터가 null이 올 수도 있고 아닐 수도 있는 경우에는 Optional.ofNullbale로 생성할 수 있습니다. 그리고 이후에 orElse 또는 orElseGet 메소드를 이용해서 값이 없는 경우라도 안전하게 값을 가져올 수 있습니다.

### orElse문과 orElseGet 문 차이점

- orElse는 null이든지 말든지 항상 불립니다.
- orElseGet은 null일 때만 불립니다.

### orElse문

![Alt text](/assets/img/2023-09-17/image-3.png)

![Alt text](/assets/img/2023-09-17/image-4.png)

- orElse문은 optional이 null일 때도 null이 아닌 경우에도 2번 다 호출됨을 알 수 있습니다.

### orElseGet 문

![Alt text](/assets/img/2023-09-17/image-5.png)

![Alt text](/assets/img/2023-09-17/image-6.png)

- orElseGet문은 optional이 null일 때에만 실행됨을 알 수 있습니다.

참고 자료 1 : https://engkimbs.tistory.com/646

참고 자료 2 : https://cfdf.tistory.com/34

참고 자료 3 : http://www.tcpschool.com/java/java_stream_optional

참고 자료 4 : https://mangkyu.tistory.com/70

### 리팩토링힌 코드 잘못된 점
- 제가 원래 의도대로 설계한 로직은 WebClient 에러가 발생하면 null을 반환하여 other 스크랩으로 저장되는 로직이었습니다.
- WebClient 에러가 발생하지 않는다면, 반환한 스크랩으로만 저장하는 로직입니다.
- 그러나, `.orElse(otherService.saveOther(new WebClientBodyResponse(), user, pageUrl))` 코드가 WebClient가 응답한 값인 crawlingResponse가 null이 아닌 경우에도 실행되므로 `articleService.saveArticle(response, user, pageUrl)`과 `otherService.saveOther(new WebClientBodyResponse(), user, pageUrl)` 총 2번 스크랩이 저장됩니다.

```java
return Optional.ofNullable(crawlingResponse)
            .map(response -> {
                String type = response.getType();
                switch (type) {
                    case "video":
                        return videoService.saveVideo(response, user, pageUrl);
                    case "article":
                        return articleService.saveArticle(response, user, pageUrl);
                    case "product":
                        return productService.saveProduct(response, user, pageUrl);
                    case "place":
                        return placeService.savePlace(response, user, pageUrl);
                    default:
                        return otherService.saveOther(response, user, pageUrl);
                }
            })
            .orElse(otherService.saveOther(new WebClientBodyResponse(), user, pageUrl));
```

- 따라서 orElse문이 아닌 orElseGet을 사용해서 원래의 의도로 null이 아닌 경우 1번만 동작하도록 변경해주도록 하겠습니다.

```java
return Optional.ofNullable(crawlingResponse)
            .map(response -> {
                String type = response.getType();
                switch (type) {
                    case "video":
                        return videoService.saveVideo(response, user, pageUrl);
                    case "article":
                        return articleService.saveArticle(response, user, pageUrl);
                    case "product":
                        return productService.saveProduct(response, user, pageUrl);
                    case "place":
                        return placeService.savePlace(response, user, pageUrl);
                    default:
                        return otherService.saveOther(response, user, pageUrl);
                }
            })
            .orElseGet(() -> otherService.saveOther(new WebClientBodyResponse(), user, pageUrl));
```

---

# 2️⃣ 본론 2
## 🚨 문제가 생긴 원인과 앞으로의 대비책
- 다행히도 Dev 서버(개발 환경)과 Prod 서버(운영 환경)을 따로 분리했기 때문에 실제로 동작을 점검해보면서 미처 놓쳤던 부분을 확인할 수 있었고, 운영환경에는 영향이 없었습니다.

### 1. 문제 원인 : 테스트 코드가 모든 로직을 커버하지 못함
- 하지만, 테스트 코드를 모두 통과한 상태였기 때문에 창피한 부분이지만 테스트 코드가 모든 로직을 커버하지는 못할 만큼 작성되어 있음을 알게 되었습니다.
- 실제로 테스트 코드를 살펴봐도 ScrapService 부분에서 scrapService.saveScraps() 메소드 부분은 아래의 코드만 있었습니다.
- 아래의 코드는 null인 경우만 확인했기 때문에 not null인 경우는 정상적으로 확인할 수 없는 것이었습니다.

```java
@Test
void should_other_type_of_scrap_is_saved_When_webClientService_crawlingItem_returns_null() throws ParseException {
    // webClientService.crawlingItem()이 null을 반환할 때, Other 타입의 Scrap이 저장되는지 확인
    //given
    memoRepository.deleteAll();
    scrapRepository.deleteAll();

    BDDMockito.when(webClientService.crawlingItem("http://localhost:123", pageUrl))
            .thenReturn(null);

    User user = userRepository.findById(1L).get();

    //when
    //then
    assertThat(scrapService.saveScraps(user, pageUrl)).isInstanceOf(Other.class);
    assertThat(scrapRepository.findByPageUrlAndUserAndDeletedDateIsNull(pageUrl, user)
            .isPresent()).isTrue();
}
```

- 따라서 not null인 경우의 테스트를 추가해주겠습니다.

```java
@Test
void should_one_article_scrap_is_saved_When_webClientService_crawlingItem_returns_article() throws ParseException {
    // webClientService.crawlingItem()이 type을 article로 반환할 때, Article 타입의 Scrap이 1개만 저장되는지 확인
    //given
    memoRepository.deleteAll();
    scrapRepository.deleteAll();

    WebClientBodyResponse webClientBodyResponse = new WebClientBodyResponse().builder()
            .title("title")
            .type("article")
            .build();

    BDDMockito.when(webClientService.crawlingItem("test", pageUrl))
            .thenReturn(webClientBodyResponse);

    User user = userRepository.findById(1L).get();

    //when
    //then
    assertThat(scrapService.saveScraps(user, pageUrl)).isInstanceOf(Article.class);
    assertThat(scrapRepository.count()).isEqualTo(1);
}
```
- 만약에 이 테스트를 넣고 리팩토링 코드를 orElse문을 그대로 사용했다면, 테스트에서 통과하지 못해서 dev 서버에 반영하지 못했을 것입니다.

![Alt text](/assets/img/2023-09-17/image-7.png)

- 그리고 orElse문이 아닌 orElseGet으로 변경했다면, 성공적으로 저장됨을 알 수 있습니다.

![Alt text](/assets/img/2023-09-17/image-8.png)

따라서 앞으로는 이러한 문제가 발생하지 않도록 테스트 코드를 예외 처리뿐만 아니라 정상적인 로직 및 로직을 체계적으로 세워서 발생할 수 있는 다양한 상황을 테스트 코드에 반영해야겠다고 다짐하게되었습니다!!

### 2. 문제 원인 : PR 단위가 크다.
![Alt text](/assets/img/2023-09-17/image-9.png)
- 리팩토링 과정에서 WebClient 로직뿐만 아니라, Optional 처리까지 여러 가지 코드를 수정하다 보니, WebClient 로직만 테스트 코드를 작성해서 Optional과 같이 문법적인 요소는 크게 신경쓰지 못한 것 같습니다. 따라서, 앞으로는 리팩토링 과정도 세분화해서 하나의 PR이 아닌 여러 개의 PR로 나누어서 로직 변경, 문법 변경과 같이 나눈 뒤에 그에 맞는 테스트 코드와 에러 처리를 추가해야겠다는 생각을 하게 되었습니다.

---

# 3️⃣ 결론
- (위에서 언급한 이야기) 앞으로는 이러한 문제가 발생하지 않도록 테스트 코드를 예외 처리뿐만 아니라 정상적인 로직 및 로직을 체계적으로 세워서 발생할 수 있는 다양한 상황을 테스트 코드에 반영해야겠다고 다짐하게되었습니다!!
- (위에서 언급한 이야기) 리팩토링 과정에서 WebClient 로직뿐만 아니라, Optional 처리까지 여러 가지 코드를 수정하다 보니, WebClient 로직만 테스트 코드를 작성해서 Optional과 같이 문법적인 요소는 크게 신경쓰지 못한 것 같습니다. 따라서, 앞으로는 리팩토링 과정도 세분화해서 하나의 PR이 아닌 여러 개의 PR로 나누어서 로직 변경, 문법 변경과 같이 나눈 뒤에 그에 맞는 테스트 코드와 에러 처리를 추가해야겠다는 생각을 하게 되었습니다.
- 멘토님이 '회사에서 신입이 실수를 했는데, 이로 인해서 큰 장애가 서비스 전체에 영향을 미쳤다. 그러나, 이러한 원인은 신입에게 서비스 전체에 영향을 줄 수 있는 권한을 부여한 시스템의 잘못이지 신입의 잘못이 아니라고 말씀해주셨습니다. 그리고 회사에 코드 리뷰 시간을 늘리고, 실제 운영 환경에 배포하기 전에 승인받는 시스템으로 변경되었다'고 말씀해주셨습니다.
- 문법의 실수로 에러가 발생해서 약간 부끄럽지만, 언제나 실수나 에러가 발생할 수 있는 만큼 부끄러움을 이겨내고 기록으로 남겨 놓음으로써 실수의 원인을 바로 직면하여 파악하고 시스템을 점검하고 대책을 마련하여 해결해나가는 것이 중요함을 깨닫게 되었습니다!!
- 마지막으로 운영 환경과 개발 환경을 나누어서 테스트를 여러 번 하는 것처럼 테스트 코드로는 모든 상황을 다 대비하고 알아낼 수 없기 때문에 테스트 환경을 많이 마련하는 것이 매우 매우 유용하고 실제 서비스에서는 필요함을 알았습니다!
