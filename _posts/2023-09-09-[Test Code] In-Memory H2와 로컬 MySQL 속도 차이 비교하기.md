---
title: "[Test Code] 테스트 DB의 In-Memory H2와 로컬 MySQL 속도 차이 비교하기"
date: 2023-09-09
categories: [Test Code, H2, MySQL]
---

# 0️⃣ 서론

지난 번에 "**In-Memory H2 사용해서 Spring Boot 테스트 코드 작성하기**" 라는 글의 제목으로 **테스트 DB를 선택**하는 것과 관련하여 글을 작성했습니다.

<a href="https://handayeon-coder.github.io/posts/In-Memory-H2-사용해서-Spring-Boot-테스트-코드-작성하기/"> https://handayeon-coder.github.io/posts/In-Memory-H2-사용해서-Spring-Boot-테스트-코드-작성하기/ </a>

이 때, MySQL에서 In-Memory H2로 변경한 가장 큰 이유가 '**속도**'라는 점이었습니다!

그러면, **과연 MySQL과 In-Memory H2 DB의 속도는 얼마나 차이가 날까??** 라는 궁금점이 들었습니다.

![Alt text](/assets/img/2023-09-09/image-2.jpg)


이전까지 작성된 16개의 테스트를 각각 다른 DB를 사용해서 돌려보며 속도 차이가 얼마 차이 날지 비교해보았습니다.


# 1️⃣ 본론 1
## 테스트 DB 변경하기 🔄

### 1. In-Memory H2에서 MySQL로 변경하기

### 1-1. application-test.yml 파일을 변경

- In-Memory H2 테스트 파일 설정
```yml
spring:
  datasource :
    url: jdbc:h2:mem:test;
    driverClassName: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect
```

- 로컬 MySQL 테스트 파일 설정 

```yml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop
    database: mysql
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect

  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/실제 DB명
    username: 실제 유저명
    password: 실제 비밀번호
```

### 1-2. 기존의 파일 중 H2와 MySQL 서로 다른 SQL 구문 변경
- 저는 `@Sql(scripts = "/truncate.sql", executionPhase = ExecutionPhase.AFTER_TEST_METHOD)`에서 scrap과 users라는 테이블을 truncate하고 있습니다.
- 이때, H2는 외래키 제약을 제거하기 위해서 `SET REFERENTIAL_INTEGRITY FALSE;`를 사용하지만, MySQL은 `SET FOREIGN_KEY_CHECKS = 0;`을 사용하므로 해당 SQL 구문을 변경해주겠습니다.

<br/>

- In-Memory H2에서 사용하는 truncate.sql 구문 
```sql
SET REFERENTIAL_INTEGRITY FALSE;
TRUNCATE TABLE scrap;
TRUNCATE TABLE users;
SET REFERENTIAL_INTEGRITY TRUE;
```

- 로컬 MySQL에서 사용하는 truncate.sql 구문 
```sql
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE scrap;
TRUNCATE TABLE users;
SET FOREIGN_KEY_CHECKS = 1;
```

---

- 아래의 예시는 현재 작성되어 있는 테스트 코드 16개 중에서 2개입니다. 

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Sql(scripts = "/truncate.sql", executionPhase = ExecutionPhase.AFTER_TEST_METHOD)
@Sql(scripts = "/setup.sql", executionPhase = ExecutionPhase.BEFORE_TEST_METHOD)
public class ScrapControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithCustomMockUser
    public void should_Scrap_When_KeywordExistSInTitle() throws Exception {
        mockMvc.perform(get("/v1/scraps/search")
                        .param("keyword", "맥북")
                        .param("page", "0")
                        .param("size", "10")
                        .header("X-AUTH-TOKEN", "aaaaaaa"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.data.content[0].title").value("Coupang의 맥북 상품"));
    }

    @Test
    @WithCustomMockUser
    public void should_Scrap_When_KeywordExistSInDescription() throws Exception {
        mockMvc.perform(get("/v1/scraps/search")
                        .param("keyword", "16인치")
                        .param("page", "0")
                        .param("size", "10")
                        .header("X-AUTH-TOKEN", "aaaaaaa"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.data.content[0].title").value("Coupang의 맥북 상품"));
    }
}

```

# 2️⃣ 본론 2

## 속도 차이 비교해보기 🔎
- 먼저, DB 수행 시간을 비교하기 위해서 최대한 동일한 조건으로 맞춰주었습니다.
1. 디바이스 : MackBook Pro 14(2023년 모델) Apple M2 Pro, 메모리 32GB, SSD 512GB
2. Intellij IDEA 2023.2
3. MySQL 버전 : 8.0.33
4. H2 버전 : 2.2.222 (2023년 9월 기준 최신 버전)
5. Spring Boot 버전 : 2.7.13, Java 버전 : 11

![Alt text](/assets/img/2023-09-09/image.png)

![Alt text](/assets/img/2023-09-09/image-1.png)

|테스트 명|In-Memory H2 소요 시간|MySQL 소요 시간|속도 증가/감소율|
|---|---|---|---|
|전체 테스트|1.35s|1.74s|-22.41%|
|should_Return_two_videos_When_Searching_for_a_keyword_that_corresponds_to_two_videos()|460ms|535ms|-14.02%|
|should_Products_When_Keyword_exists_in_title()|251ms|281ms|-10.68%|
|should_Products_When_Keyword_exists_in_description()|133ms|52ms|155.77%|
|should_Return_one_product_When_Searching_for_a_keyword_that_corresponds_to_one_product()|31ms|42ms|-26.19%|
|should_videos_are_sorted_in_created_date_When_searching_for_multiple_scraps()|46ms|72ms|-36.11%|
|IfUserExistsThenGetUserInfoReturnsSuccess()|20ms|49ms|-59.18%|
|should_Scrap_When_KeywordExistSInDescription()|53ms|72ms|-26.39%|
|should_Scrap_When_KeywordExistSInTitle()|19ms|36ms|-47.22%|
|should_only_article_is_searched_When_searching_for_keywords_that_are_also_in_other_scraps()|37ms|189ms|-80.42%|
|contextLoads()|3ms|6ms|-50%|
|should_Return_zero_other_When_Searching_for_a_keyword_in_not_existent_other()|33ms|55ms|-40%|
|IfExistentMemberDeletesNonExistentScrapThenReturnNotFoundException()|62ms|76ms|-18.42%|
|IfNonExistentMemberDeletesScrapThenReturnNotFoundException()|14ms|27ms|-48.15%|
|IfExistentMemberDeletesOneScrapThenReturnSuccess()|95ms|121ms|-21.49%|
|should_Return_zero_article_When_Searching_for_a_keyword_that_not_corresponds_to_article()|18ms|31ms|-41.94%|
|평균 |||-24.18%|

|계층별 테스트|In-Memory H2 소요 시간|MySQL 소요 시간|속도 증가/감소율|
|---|---|---|---|
|VideoServiceTest|460ms|535ms|-14.02%|
|ProductServiceTest|31ms|42ms|-26.19%|
|OtherServiceTest|33ms|55ms|-40%|
|ScrapServiceTest|171ms|224ms|-23.66%|
|ArticleServiceTest|18ms|31ms|-41.94%|
|평균 |||-29.16%|
|ProductControllerTest|384ms|333ms|15.32%|
|VideoControllerTest|46ms|72ms|-36.11%|
|UserControllerTest|20ms|49ms|-59.18%|
|ScrapControllerTest|37ms|189ms|-80.42%|
|OtherControllerTest|76ms|98ms|-22.45%|
|ArticleControllerTest|18ms|31ms|-41.94%|
|평균 |||-37.46%|

따라서 In-Memory H2를 사용했을 때가 MySQL을 사용했을 때보다 평균적으로 24.18%가 더 빠르다는 것을 알 수 있습니다.

현재는 테스트의 개수가 16개밖에 없어서 전체 테스트를 돌렸을 때 시간 차이가 크지 않지만 앞으로 테스트의 개수가 늘어날 수록 시간 차이가 크게 날 수 있을 것이라고 예상해볼 수 있었습니다.

---

## 속도 차이가 나는 이유 🐢 🐰
그러면 이렇게까지 속도 차이가 나는 이유는 무엇일까요??

바로 In-Memory라는 H2 앞에 계속 붙었던 이름 때문입니다.

> 인메모리 데이터베이스의 속도: H2 인메모리 데이터베이스는 디스크 I/O 없이 메모리에서 데이터를 처리하기 때문에 일반적으로 디스크 기반의 데이터베이스보다 빠릅니다. 로컬 MySQL은 디스크에 데이터를 쓰고 읽어오는 작업이 필요하므로 이러한 작업이 추가 속도 지연을 일으킬 수 있습니다.
출처 - ChatGPT

저는 대학교 3학년 2학기 때 시스템 프로그래밍을 배우면서 교수님께서 `메모리`는 내 앞에 있는 책상에 놓여있는 책들이고, `디스크`는 집 앞에 도서관이라고 설명해주신 기억이 떠올랐습니다.

즉, 메모리를 사용하면 훨씬 빠른 속도로 메모리를 사용하는 H2를 사용할 수 있습니다.

![Alt text](/assets/img/2023-09-09/image-3.png)

사진 출처 : https://m.blog.naver.com/sjc02183/221998493348

# 3️⃣ 결론
- 빠른 속도를 위해서 In-Memory H2로 변경하면서, 얼마나 속도 차이가 날 지 궁금하면서 테스트 코드를 작성하게 되었는데 실제로 속도 차이를 직접 비교해보고 제법 속도 차이가 나는 모습을 보고 DB 변경이 의미가 있는 변경이었구나라고 생각하게 되었습니다!
-  아직까지는 작성된 테스트 코드 자체가 많이 있지 않아서, 속도 차이가 크게 나보이지는 않지만, 앞으로 테스트 코드를 많이 작성할 수록 더욱더 In-memory H2의 진가가 날 것이라고 기대하고 있습니다!
- 그리고 위에서 In-Memory H2보다 MySQL이 더 빨랐던 메소드가 1개 있는데 왜 그런지 파악해보고 싶다는 생각도 들었습니다.