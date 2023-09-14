---
title: "[WebClient] WebClient 사용해서 외부 API 호출하기"
date: 2023-09-14
categories: [WebClient]
---

# 0️⃣ 서론

저희 서비스는 크롤링하는 서버를 EC2와 분리해서 서비스를 분리시켰기 때문에, Spring Boot(AWS EC2)에서 AWS Lambda 서버를 호출하는 형식으로 외부 API를 호출하고 있습니다.

따라서, 간략하게 AWS EC2에서 AWS Lambda를 호출하는 흐름을 아래의 그림과 같이 나타내었습니다.

![Alt text](/assets/img/2023-09-14/image.png)

그러면 Spring Boot에서는 어떻게 외부의 API를 호출시킬까요??

저희는 **WebClient**라는 외부 호출 방식를 사용하였습니다.

그러면 도대체 WebClient가 무엇이길래 이거를 선택했고 어떻게 사용할지 소개해드리겠습니다.

---

# 1️⃣ 본론 1

## ✔️ 외부 호출 방식 결정하기

먼저, 'Spring으로 외부 API 호출하기'라고 구글에 검색했을 때 가장 많이 나오는 방식은 'RestTemplate'과 'WebClient'입니다.

그러면 두 가지 방식은 어떠한 차이가 있을까요??

### RestTemplate
- Rest API를 호출하고, 응답을 제어할 수 있는 스프링에서 제공하는 클래스입니다.
- 스프링 3.0에서부터 지원하는 RestTemplate은 HTTP 통신에 유용하게 쓸 수 있는 템플릿입니다.
- REST 서비스를 호출하도록 설계되어 HTTP 프로토콜의 메서드 (GET, POST, DELETE, PUT)에 맞게 여러 메서드를 제공합니다.
- 통신을 단순화하고 **RESTful 원칙**을 지킵니다.
- **멀티쓰레드 방식**을 사용합니다.
- **Blocking 방식**을 사용합니다.

그러나, 스프링 5.0에서 WebClient가 나오면서 WebClient를 공식 문서에서는 추천해주고 있다.
(아직은 소문이지만, RestTemplate이 deprecated 될 수도 있다는 소리를 들어서, 이전에 RestTemplate를 사용한 것이 아니라 새로운 프로젝트를 하는 저의 경우에는 WebClient가 더 좋다고 생각했습니다.)

> NOTE: As of 5.0 this class is in maintenance mode, with only minor requests for changes and bugs to be accepted going forward. Please, consider using the org.springframework.web.reactive.client.WebClient which has a more modern API and supports sync, async, and streaming scenarios.

### WebClient
- 스프링 5.0에서 추가된 인터페이스입니다.
- **싱글 스레드 방식**을 사용합니다.
- **Non-Blocking 방식**을 사용합니다.
- **JSON, XML**을 쉽게 응답받습니다.

따라서 가장 큰 차이점은 *Non-Blocking 여부*와 *비동기화 여부*입니다.

||RestTemplate|WebClient|
|---|---|---|
|Non-Blocking|불가능|가능|
|비동기화|불가능|가능|

- 즉, WebCLient는 시스템을 호출한 직후에 프로그램으로 제어가 다시 돌아와서 시스템 호출의 종료를 기다리지 않고 다음 동작을 진행합니다. 따라서 호출한 시스템의 동작을 기다리지 않고 동시에 다른 작업을 진행할 수 있습니다.

- 그리고 WebClient는 block()을 사용해서 Non-Blocking과 Blocking을 자유롭게 변경할 수 있습니다.

- 따라서 이러한 장점을 많이 가지고 있고, 공식 문서에서 추천하는 WebClient를 저희 프로젝트에 도입하게 되었습니다.

- 현재는 외부 API를 호출하고 나서 외부 API의 응답값을 DTO로 변환하여 DB에 저장하는 방식으로 진행하고 있어서 WebClient를 Blocking으로 사용하고 있습니다... 그래서 큰 장점을 활용하지 못하고 있지만, 앞으로 사용자가 많아진다면 아키텍처 구조를 변경하여 WebClient를 Non-Blocking 구조로 변경할 예정입니다.!

참고 자료 1 : <a href="https://velog.io/@chlwogur2/Spring-외부-API-호출-로직에-관해"> https://velog.io/@chlwogur2/Spring-외부-API-호출-로직에-관해 </a>

참고 자료 2 : <a href="https://tecoble.techcourse.co.kr/post/2021-07-25-resttemplate-webclient/"> https://tecoble.techcourse.co.kr/post/2021-07-25-resttemplate-webclient/ </a>

## 💼 WebClient 사용해서 외부 API 호출하기
### 1.  의존성 추가하기
```yml
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

### 2. WebClient 생성하기

- Builder를 사용하여 WebClient를 생성하겠습니다.
- 이때, 외부 API의 주소를 baseUrl에 AWS Lambda의 EndPoint(API GATEWAY)를 작성해주었습니다.

```java
WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();
```

### 3. 외부 API 호출하기
- WebClient로 외부 API를 Post 메서드를 사용해서 호출해줍니다.

```java
Map<String, Object> bodyMap = new HashMap<>();
bodyMap.put("url", pageUrl);

WebClientResponse webClientResponse = webClient.post()
                    .bodyValue(bodyMap)
                    .retrieve()
                    .bodyToMono(WebClientResponse.class)
                    .block();
```

- 위에 같이 webClient.post().bodyValue(bodyMap)을 사용하면 아래의 postman처럼 보내는 것과 동일한 형태입니다.

![Alt text](/assets/img/2023-09-14/image-2.png)

- **block()** 을 이용해서 Non-Blocking 형태가 아닌 **Blocking 형태로 변경**할 수도 있습니다.
    - 현재는 외부 API를 호출하고 나서 외부 API의 응답값을 DTO로 변환하여 DB에 저장하는 방식으로 진행하고 있어서 blocking으로 사용했습니다.
- **retrive()** 는 CleintResponse 개체의 body를 받아 디코딩하고 사용자가 사용할 수 있도록 미리 만든 개체를 제공하는 간단한 메소드 입니다.
- retrive() 를 사용할 때는, toEntity(), bodyToMono(), bodyToFlux() 이렇게 response를 받아올 수 있습니다.
    - bodyToFlux, bodyToMono 는 가져온 body를 각각 Reactor의 Flux와 Mono 객체로 바꿔줍니다.
    - 이때, mono는 외부 서비스에 요청을 할 때 `최대 하나의 결과를 예상`할 때 **Mono**를 사용해야 합니다.
    - 외부 서비스 호출에서 `여러 결과`가 예상되는 경우 **Flux**를 사용해야 합니다.
    - 저희 서비스는 하나의 결과가 나오기 때문에, 응답 결과를 **bodyToMono**로 받아오겠습니다.

WebClient 사용방법 참고 자료 : <a href="https://thalals.tistory.com/379"> https://thalals.tistory.com/379 </a>

Mono, Flux 참고 자료 : <a href="https://recordsoflife.tistory.com/799"> https://recordsoflife.tistory.com/799 </a>

### 4. 외부 API 에러 처리하기

외부 API의 응답 status가 400대, 500대 에러가 발생할 경우, RuntimeException를 발생시켜서 null을 반환하도록 하였습니다.

```java
WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();

Map<String, Object> bodyMap = new HashMap<>();
bodyMap.put("url", pageUrl);

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
} catch (Exception e) {
    return null;
}
```

에러 참고 자료 : <a href="https://dkswnkk.tistory.com/708"> https://dkswnkk.tistory.com/708 </a>

---

# 2️⃣ 본론 2
## 외부 API 호출 응답값 로직 변경 이유 (외부 API 호출하여 받은 JSON 값을 DTO로 변환하기)

이전에는 Map<String, Object> response로 응답값을 받았습니다. 

```java
WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();

Map<String, Object> bodyMap = new HashMap<>();
bodyMap.put("url", pageUrl);

Map<String, Object> response = webClient.post()
        .bodyValue(bodyMap)
        .retrieve()
        .bodyToMono(Map.class)
        .block();

JSONParser jsonParser = new JSONParser();

Object obj = jsonParser.parse(response.get("body").toString());

JSONObject jsonObject = (JSONObject) obj;
```


따라서, JSONObject에서 각각의 속성값(title, thumbnail_url, descrption 등)을 가져와서 Object를 각각의 속성값에 맞는 타입(string, long 등)으로 변환해줘야 했습니다.

이 과정에서 만약에 null이면 타입을 변환할 때 에러가 발생하므로 null인지 파악하는 코드도 추가해줘야 했습니다.

```java
public Article saveArticle(JSONObject crawlingResponse, User user, String pageUrl) {

    Article article = Article.builder().user(user).pageUrl(pageUrl)
            .title(Optional.ofNullable(crawlingResponse.get("title")).map(Object::toString)
                    .orElse(null))
            .thumbnailUrl(Optional.ofNullable(crawlingResponse.get("thumbnail_url"))
                    .map(Object::toString).orElse(null))
            .description(Optional.ofNullable(crawlingResponse.get("description"))
                    .map(Object::toString).orElse(null))
            .author(Optional.ofNullable(crawlingResponse.get("author")).map(Object::toString)
                    .orElse(null))
            .authorImageUrl(Optional.ofNullable(crawlingResponse.get("author_image_url"))
                    .map(Object::toString).orElse(null))
            .blogName(
                    Optional.ofNullable(crawlingResponse.get("blog_name")).map(Object::toString)
                            .orElse(null))
            .publishedDate(Optional.ofNullable(crawlingResponse.get("published_date"))
                    .map(Object::toString).map(Long::parseLong).map(
                            TimeService::fromUnixTime).orElse(null))
            .siteName(
                    Optional.ofNullable(crawlingResponse.get("site_name")).map(Object::toString)
                            .orElse(null)).build();

    return articleRepository.save(article);
}
```

- 앞으로 속성값이 늘어날 때마다 null 에러 처리 및 타입 변경을 일일히 해줘야 했습니다.
- 그리고 JSONParser jsonParser = new JSONParser();는 decrepted되었다고 합니다. 
- 따라서 JSONObject를 사용하지 않기로 하였습니다.

JSONParser 관련 자료 : <a href="https://myeongju00.tistory.com/77"> https://myeongju00.tistory.com/77 </a>

---

따라서 ObjectMapper를 이용해서 원하는 DTO로 변환하도록 하였습니다!

### 외부 API 호출하여 받은 JSON 값을 DTO로 변환하기

현재 외부 API인 AWS Lambda에서는 아래의 값을 응답 Body로 반환하고 있습니다.

```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"type\": \"place\", \"page_url\": \"https://map.kakao.com/?map_type=TYPE_MAP&itemId=\", \"site_name\": \"KakaoMap\", \"lat\": 37.50359, \"lng\": 127.044848, \"title\": \"서울역\", \"address\": \"서울\", \"phonenum\": \"1234-1234\", \"zipcode\": \"12345\", \"homepage\": \"https://www.seoul.co.kr\", \"category\": \"지하철\"}"
}
```

위의 구조를 큰 형태로 보면 statusCode, headers, body로 이루어져 있음을 알 수 있습니다.

아래와 같은 DTO로 응답을 받을 수 있습니다.

```java
@Getter
@NoArgsConstructor
public class WebClientResponse {

    private String statusCode;
    private WebClientHeaderResponse headers;
    private String body;
}
```

그리고 body는 아래의 WebClientBodyResponse 형태를 띄고 있습니다.

```java
@Getter
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class) //응답값이 python의 snake case이므로 아래의 속성값을 snake case로 변경해줍니다.
@NoArgsConstructor
public class WebClientBodyResponse {

    // 공통 부분
    private String type;
    private String description;
    private String pageUrl;
    private String siteName;
    private String thumbnailUrl;
    private String title;

    // Video 부분
    private String channelImageUrl;
    private String channelName;
    private String embedUrl;
    private Long playTime;
    private Long watchedCnt;
    private Long publishedDate; // Video, Article 공통 부분

    // Article 부분
    private String author;
    private String authorImageUrl;
    private String blogName;

    // Product 부분
    private String price;

    // Place 부분
    private String address;

    @JsonProperty("lat")
    private BigDecimal latitude;

    @JsonProperty("lng")
    private BigDecimal longitude;

    @JsonProperty("phonenum")
    private String phoneNumber;
    private String zipCode;

    @JsonProperty("homepage")
    private String homepageUrl;
    private String category;
}

```

따라서 아래의 .bodyToMono(WebClientResponse.class)에서 외부 API 호출하여 받은 JSON 값을 WebClientResponse DTO로 변환해주고.
ObjectMapper를 통해서 String으로 들어온 WebClientResponse의 body부분을 WebClientBodyResponse DTO로 변환해주는 작업을 수행해주면 됩니다.

```java
WebClient webClient = WebClient.builder().baseUrl(crawlingApiEndPoint).build();

Map<String, Object> bodyMap = new HashMap<>();
bodyMap.put("url", pageUrl);

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
```

- 이때 유의할 점은 `objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);`을 사용했다는 점입니다.
- json을 Dto로 변환하는 과정에서 맵핑되지 않는 속성이 있는 경우 오류를 발생시키지 않고 서로 동일한 속성값만 변환해줍니다. 

objectAmpper 관련 참고 자료 : <a href="https://tomining.tistory.com/191"> https://tomining.tistory.com/191 </a>

```java
public Article saveArticle(WebClientBodyResponse crawlingResponse, User user, String pageUrl) {

    Article article = Article.builder().user(user).pageUrl(pageUrl)
            .title(crawlingResponse.getTitle())
            .thumbnailUrl(crawlingResponse.getThumbnailUrl())
            .description(crawlingResponse.getDescription())
            .author(crawlingResponse.getAuthor())
            .authorImageUrl(crawlingResponse.getAuthorImageUrl())
            .blogName(crawlingResponse.getBlogName())
            .publishedDate(TimeService.fromUnixTime(crawlingResponse.getPublishedDate()))
            .siteName(crawlingResponse.getSiteName()).build();

    return articleRepository.save(article);
}
```

따라서 이전에 방식보다 속성값의 타입을 따로 변환해주지 않아도 되고 null 처리도 안해도 되어서 간편해졌습니다!

---

# 3️⃣ 결론

- 외부 API를 호출하는 작업은 크게 오래 걸리지 않았지만, 응답 값을 받고 이를 저희 서비스에 맞는 형태로 변환해주기 위해서는 많은 작업과 시행착오가 있었습니다.
- 그러나, 이전보다 더 에러 처리부분에서 단단해지고 코드가 간략해진 모습을 보고 refactoring의 중요성을 다시 한 번 느끼게 된 것 같습니다.
- 위에서도 언급했지만, 현재는 외부 API를 호출하고 나서 외부 API의 응답값을 DTO로 변환하여 DB에 저장하는 방식으로 진행하고 있어서 WebClient를 큰 장점인 Non-Blocking을 활용하지 못하고 있지만, 앞으로 사용자가 많아진다면 아키텍처 구조를 변경하여 WebClient를 Non-Blocking 구조로 변경해야 제대로된 WebClient를 사용할 수 있겠다는 생각을 했습니다!
- 이전에 WebClient를 사용할 때에는 응답값의 구조를 제대로 살펴보지 않아서 Dto 변환이 어려워서 MAP 구조를 사용했는데 이번 계기로 외부 API 응답값을 더 자세하게 살펴보고 이해하게 되어 더 적합한 refactoring이 가능했던 것 같습니다.☺️