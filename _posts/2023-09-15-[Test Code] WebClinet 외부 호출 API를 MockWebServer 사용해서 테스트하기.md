---
title: "[Test Code] WebClinet 외부 호출 API를 MockWebServer 사용해서 테스트하기"
date: 2023-09-14
categories: [Test Code, WebClient, MockWebServer]
---

# 0️⃣ 서론

이전 게시글에서 '*WebClient를 사용해서, 외부 API를 호출하기*'라는 글을 작성했습니다.

<a href="https://velog.io/@da_na/WebClient-WebClient-사용해서-외부-API-호출하기"> https://velog.io/@da_na/WebClient-WebClient-사용해서-외부-API-호출하기 </a>

저희 서비스는 크롤링 서버를 따로 분리했기 때문에 WebClient를 사용해서 Spring Boot(AWS EC2)가 외부 API인 크롤링 서버를 호출하는 방식으로 되어 있습니다. 

따라서 아래의 WebClient 서비스 로직은 Spring Boot가 외부 API 크롤링 서버를 호출하는 로직입니다.

```java
@Service
public class WebClientService {

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
}
```

이전의 테스트 코드를 도입하기 전까지는 POSTMAN을 사용하여 **직접 외부 서버와 통신**하고 값을 확인했습니다.

따라서, 테스트가 외부 서버에 매우 **의존적**이었고 서버를 직접 호출하기 때문에 **테스트 시간이 오래 걸렸습니다**.

그리고 이전 게시글의 마지막에 WebClient를 리팩토링하는데, WebClient로 부터 받은 값을 변환하는 과정을 변경할 때에도 계속 외부 서버와 통신하는 것도 계속 동일한 값을 받는 데에 외부 서버를 호출하는 것은 비효율적이라고 생각했습니다.

더 나아가, 400번 대, 500번 대의 외부 서버 에러를 발생시키려면 일부러 에러를 요청하는 크롤링 웹 페이지 URL을 찾아서 보내줘야 해서 에러 처리하기가 어려웠습니다.

그러면 직접 외부 서버와 통신하는 방법 말고는 다른 방법으로 테스트하는 방법은 없을지 고민하다가, MockWebServer를 선택하여 테스트에 도입할 수 있었습니다.

---

# 1️⃣ 본론 1
## 😎 MockWebServer가 무엇인가?
Square 팀에서 만든 MockWebServer는 **HTTTP Request를 받아서 Response를 반환하는 간단하고 작은 웹서버**입니다.

WebClient를 사용하여, Http를 호출하는 메서드의 테스트 코드를 작성할때 이 MockWebServer를 호출하게 함으로써 쉽게 테스트 코드를 작성할 수 있습니다.

그리고 실제로 Spring Team도 MockWebServer를 사용하여 테스트하라고 권장한다고 합니다.

MockWebServer 공식 문서 : <a href="https://github.com/square/okhttp/tree/master/mockwebserver"> https://github.com/square/okhttp/tree/master/mockwebserver </a>

참고 자료 : <a href="https://www.devkuma.com/docs/mock-web-server/"> https://www.devkuma.com/docs/mock-web-server/ </a>

Spring Team 출처 : <a href="https://github.com/spring-projects/spring-framework/issues/19852#issuecomment-453452354"> https://github.com/spring-projects/spring-framework/issues/19852#issuecomment-453452354 </a>

## 💼 MockWebServer 사용해서 외부 서버를 Mocking하기

### 1. MockWebServer 의존성 추가하기
```yml
testImplementation 'com.squareup.okhttp3:mockwebserver'
```

### 2. 테스트 코드에 MockWebServer 추가하기

- @BeforeEach를 선언하여 각각의 테스트마다 mockWebServer.start()를 사용해서, 가짜 API 서버인 MockWebServer를 켭니다.
- @AfterEach를 선언하여 각각의 테스트마다 mockWebServer.shutdown()를 사용해서, 켰던 MockWebServer를 끕니다.

```java
@Autowired
private WebClientService webClientService;

private MockWebServer mockWebServer;

private String mockWebServerUrl;

@BeforeEach
void setUp() throws IOException {
    mockWebServer = new MockWebServer();
    mockWebServer.start();
    mockWebServerUrl = mockWebServer.url("/v1/crawling").toString();
}

@AfterEach
void terminate() throws IOException {
    mockWebServer.shutdown();
}
```

### 3. MockWebServer 사용하여 서비스 로직 테스트하기

- 여기에서는 mockWebServer.enqueue를 사용하여 mockWebServer 가짜 서버가 응답할 값을 설정해줄 수 있습니다.
- 아래의 코드에서는 성공적으로 응답하는 것을 확인하고, 응답값을 DTO로 잘 변환해주는지 확인하는 과정이기 때문에 .setResponseCode(200)으로 응답값을 200으로 설정해주었습니다. 그리고 mockScrap은 따로 private String mockScrap으로 선언해었습니다.(아래에서는 선언한 것은 생략하였습니다!)

```java
@Test 
void should_title_is_returned_When_the_webClient_server_responds_successfully() {
    //given
    mockWebServer.enqueue(new MockResponse()
            .setResponseCode(200)
            .setBody(mockScrap)
            .addHeader("Content-Type", "application/json"));

    //when
    WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(
            mockWebServerUrl, "https://www.naver.com");

    //then
    assertThat(webClientBodyResponse.getTitle()).isEqualTo("서울역");
}
```
![Alt text](/assets/img/2023-09-15/image.png)

### 4. MockServer 사용하여 400번대의 응답 코드 보내서 에러 처리 로직 테스트 하기
- 다음으로는 정상적인 응답이 아니라, 400번의 에러가 발생했을 때 서비스 로직이 에러 처리가 제대로 되고 있는지 확인하는 테스트 코드를 작성해주었습니다.
- 이때, 저는 외부 API 서버가 에러가 발생하면 null을 반환하도록 설계했기 때문에 아래의 테스트 코드는 null이 나오는지 확인하는 구조입니다.
- 이처럼 쉽게 외부 API 서버의 에러를 발생시킬 수 있어서, 에러 처리를 확인하고 더 구체적으로 작성할 수 있습니다.
```java
@Test
void should_it_returns_null_When_webClient_server_responds_unsuccessfully() {
    //given
    mockWebServer.enqueue(new MockResponse()
            .setResponseCode(400)
    );

    //when
    WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(
            mockWebServerUrl, "https://www.naver.com");

    //then
    assertThat(webClientBodyResponse).isEqualTo(null);
}
```
![Alt text](/assets/img/2023-09-15/image-1.png)


---

전체적인 MockWebServer를 이용한 테스트 코드입니다!

```java
@SpringBootTest
@ActiveProfiles("test")
public class WebClientServiceTest {

    @Autowired
    private WebClientService webClientService;

    private MockWebServer mockWebServer;

    private String mockWebServerUrl;

    private final String mockScrap = "{\n"
            + "  \"statusCode\": 200,\n"
            + "  \"headers\": {\n"
            + "    \"Content-Type\": \"application/json\"\n"
            + "  },\n"
            + "  \"body\": \"{\\\"type\\\": \\\"place\\\", \\\"page_url\\\": \\\"https://map.kakao.com/1234\\\", "
            + "\\\"site_name\\\": \\\"KakaoMap\\\", \\\"lat\\\": 37.50359439708544, \\\"lng\\\": 127.04484896895218, "
            + "\\\"title\\\": \\\"서울역\\\", \\\"address\\\": \\\"서울특별시 중구 소공동 세종대로18길 2\\\", "
            + "\\\"phonenum\\\": \\\"1522-3232\\\", \\\"zipcode\\\": \\\"06151\\\", "
            + "\\\"homepageUrl\\\": \\\"https://www.seoul.co.kr\\\", \\\"category\\\": \\\"지하철\\\"}\"\n"
            + "}\n";

    @BeforeEach
    void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
        mockWebServerUrl = mockWebServer.url("/v1/crawling").toString();
    }

    @AfterEach
    void terminate() throws IOException {
        mockWebServer.shutdown();
    }

    @Test 
    void should_title_is_returned_When_the_webClient_server_responds_successfully() {
        //given
        mockWebServer.enqueue(new MockResponse()
                .setResponseCode(200)
                .setBody(mockScrap)
                .addHeader("Content-Type", "application/json"));

        //when
        WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(
                mockWebServerUrl, "https://www.naver.com");

        //then
        assertThat(webClientBodyResponse.getTitle()).isEqualTo("서울역");
    }


    @Test
    void should_it_returns_null_When_webClient_server_responds_unsuccessfully() {
        //given
        mockWebServer.enqueue(new MockResponse()
                .setResponseCode(400)
        );

        //when
        WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(
                mockWebServerUrl, "https://www.naver.com");

        //then
        assertThat(webClientBodyResponse).isEqualTo(null);
    }
}
```

---

# 2️⃣ 본론 2

## 의존성 줄이기 ➡️ WebClientService 로직 구조 변경

- 이러한 WebClientService를 테스트하기 위한 과정에서 MockWebServer로 테스트가 가능한 유연한 구조를 변경했습니다.
- 아래는 이전의 테스트를 작성하기 전의 코드입니다.

### 이전의 구조

```java
@Service
 public class WebClientService {

     @Value("${crawling.server.post.api.endPoint}")
     private String crawlingApiEndPoint;

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
 }
```

- 이전 코드의 문제점은 WebClientService의 crawlingItem 메소드가 pageUrl만 받는 구조라는 점입니다.
    - 이러한 구조 때문에 MockWebServer로 만든 가짜 서버의 URL을 입력해주지 못해서 해당 메소드를 테스트하지 못합니다.
    - 그러면, 🤔 이때 드는 의문은 *@Value로 crawling.server.post.api.endPoint에서 값을 가져오고 있는데 여기에 값을 미리 입력해주면 되는 거 아닌가??* 라는 의문이 들게 됩니다.
    - 하지만, **MockWebServer**는 **port 번호를 테스트할 때마다 마음대로 생성**하기 때문에 미리 URL을 입력할 수 없습니다.
    - 따라서, 외부 API URL을 외부에서 주입하는 구조로 변경하여 테스트에 유연한 구조로 변경하였습니다. (현재 인프런의 김영한님 스프링 강의를 듣고 있는데 생성자 주입 DI가 떠올랐습니다 ㅎㅎ)

### MockWebServer의 Port 번호 확인하기
- MockWebServer는 port 번호를 테스트할 때마다 마음대로 생성하는지 확인하기 위해서 아래와 같이 출력하는 코드를 입력해서 테스트를 2번 실행해보았습니다.

```java
@BeforeEach
void setUp() throws IOException {
    mockWebServer = new MockWebServer();
    mockWebServer.start();
    mockWebServerUrl = mockWebServer.url("/v1/crawling").toString();

    System.out.println("mockWebServerUrl = " + mockWebServerUrl);
}
```

![Alt text](/assets/img/2023-09-15/image-2.png)
![Alt text](/assets/img/2023-09-15/image-3.png)

- 위의 사진처럼 포트 번호가 50346, 50352로 다른 것을 알 수 있습니다.

### 변경한 구조

- 아래처럼 WebClientService의 crawlingItem 메소드 파라미터로 외부 API URL을 입력받는 구조로 변경하였습니다.
- 따라서, 가짜 외부 API 서버의 URL과 실제 외부 API 서버의 URL을 모두 받을 수 있는 유연한 구조가 되었습니다!
- 해당 메소드 테스트 시 사용하는 코드 : `WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(mockWebServerUrl, "https://www.naver.com");`
```java
@Service
 public class WebClientService {

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
 }
```
---

# 3️⃣ 결론

### 1. ⛑️ **안정적이고 다양한 에러 처리 가능** 
- 해당 테스트 코드를 작성하기 전에는 직접 외부 API 서버를 호출해서 테스트를 진행했기 떄문에 에러가 나는 호출을 찾기 위해서 여러 번의 에러가 나는 페이지들을 찾아서 해당 페이지를 가지고 외부 API를 호출했습니다. 
그러나, 테스트를 준비하기 위해서 더 오랜 시간이 걸렸고, 웹 페이지 구조가 변경되면 해당 페이지가 에러가 나지 않을 수도 있는 등의 외부 서버에 너무 높은 의존도를 가지고 있었습니다. 
하지만 이번 테스트 코드를 작성하고 나니까, 훨씬 더 다양한 에러를 항상 빠르게 발생시킬 수 있었고 다양한 에러를 발생시키다 보니까 미처 놓쳤던 에러에 더 적합한 처리도 할 수 있게 되어서 더 안정적인 서비스가 가능해진 것 같습니다.

### 2. 🤸‍♀️ **테스트 하기 위해서 유연한 구조로 변경**
- 테스트를 진행하기 위해서 외부에서 주입받는 형식으로 변경하여 테스트에 유연한 구조로 변경하였는데, 유연한 구조 설계의 중요성을 다시 한 번 느끼게 된 것 같습니다.

### 3. 🤝 **테스트 코드는 약속이다.** 
- 테스트 코드를 작성하면서, 이전에는 외부 API에서 에러가 발생했을 때 따로 처리를 하지 않아서 외부 API의 에러가 발생하면 현재 프로젝트의 Spring Boot 서버(외부 API 호출한 원래의 서버)도 에러가 발생하였습니다.
- 그러나, 이번 테스트 코드로 에러 처리가 가능해지면서, 외부 API 에러를 원래의 서버에 영향을 줄이고자 null을 반환하도록 하고 Other(기타 스크랩)으로 저장되는 로직으로 변경하였습니다.
- 테스트 관련 강의를 들으면서 테스트 코드를 작성함으로써, 어떠한 행동과 로직을 수행했을 때 어떠한 결과가 나와야 한다고 문서화 및 약속이 가능하다!!라는 말이 떠올랐습니다. (왜냐하면, 테스트 코드는 모든 팀원들이 봐야하고 항상 모든 서비스 로직이 테스트 코드를 통과해야 하며, 행동에 대한 결과가 정해져있기 때문입니다.)
- 따라서 이러한 과정을 아래의 테스트 코드로 작성해놓으면서,  팀원들과 같이 이야기를 하고 로직에 대한 약속을 정할 수 있게 되었습니다.
- WebClient(외부 API 서버)가 비정상적인 응답을 준 경우 null로 반환하고 Other 스크랩으로 저장되는 로직을 수행했습니다.

```java
@Test
void should_it_returns_null_When_webClient_server_responds_unsuccessfully() {
    // webClient 서버가 정상적으로 응답을 주지 않는 경우, null로 반환되는지 확인
    //given
    mockWebServer.enqueue(new MockResponse()
            .setResponseCode(400)
    );

    //when
    WebClientBodyResponse webClientBodyResponse = webClientService.crawlingItem(
            mockWebServerUrl, "https://www.naver.com");

    //then
    assertThat(webClientBodyResponse).isEqualTo(null);
}
```

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
### 4. 🛟 **리팩토링시, 테스트 코드를 통해서 훨씬 안전하게 변경가능**

- 이전 글에서 외부 API 호출하는 로직을 리팩토링한 적이 있습니다. 이번 글의 테스트 코드와 리팩토링 글이 나누어져 있지만 원래는 리팩토링 과정에서 테스트 코드도 같이 도입된 상태였습니다.
- 이때, 느꼈던 점은 리팩토링을 할 때 테스트 코드를 작성해 놓아서 테스트 코드를 통과하면 로직이 정상적으로 돌아간다는 보장이 있기 때문에 테스트 코드를 믿으면서 안전하게 코드를 마음대로 수정할 수 있게 되었습니다.
- 그리고 테스트 코드를 통해서 응답 결과를 바로 확인할 수 있어서 실제 외부 API를 호출하지 않고도 쉽게 수정이 용이했습니다!!
- 이전 글 : <a href="https://velog.io/@da_na/WebClient-WebClient-사용해서-외부-API-호출하기"> https://velog.io/@da_na/WebClient-WebClient-사용해서-외부-API-호출하기 </a>
