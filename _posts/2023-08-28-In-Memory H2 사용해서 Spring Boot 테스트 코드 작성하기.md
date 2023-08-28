---
title: "In-Memory H2 사용해서 Spring Boot 테스트 코드 작성하기"
date: 2023-08-28
categories: [H2,Spring Boot, Test Code]
---

### PART 1. 테스트 코드 작성시, 구성할 DB 선정하기

대표적으로 스프링 부트에서 사용되는 DB는
내장형(In-Memory) `H2 데이터 베이스`와 `로컬 MySQL 데이터 베이스`가 있습니다.

이때, 여러 요인을 고려해서 두 가지의 DB 중에서 선택해보도록 하겠습니다.

저희 프로젝트는 현재 로컬 MySQL을 사용해서 데이터 베이스를 전체적으로 구성하고 있습니다.

따라서, 테스트 환경과 실제 운영 환경을 동일시하기 위해서 테스트 코드 환경을 로컬 MySQL로 구성했습니다. 

- 테스트를 돌리기 전에 반드시 항상 로컬 MySQL을 켜야하는 문제가 발생했습니다.

- 테스트 코드 실행의 속도가 느리다는 문제가 발생했습니다.

여기에서 생기는 문제점 중에서 가장 큰 문제점은 `속도 측면`이었습니다.

- Github action에서 test할 떄 비공개 레포지토리인 경우 시간당 비용이 나가게 됩니다. 그런데, 속도가 느리면 비용이 더 들게 됩니다.
- 프로젝트가 점차 커질수록 테스트의 양도 늘어나게 되는데, 속도가 느리면 테스트 소요시간이 점차 증가하게 됩니다. 그래서, 테스트로 인한 개발 시간이 지연될 수 있습니다.

따라서, 점차 프로젝트의 규모가 커질 것으로 예상되어, 로컬 MySQL이 아닌 `내장형(In-Memory) H2`를 사용하여 프로젝트의 테스트 환경을 구축하기로 하였습니다.

---
### PART 2. H2 데이터 베이스의 3가지 모드

H2 데이터 베이스는 3가지 모드로 동작할 수 있습니다. (출처 : ChatGPT)

#### 1. `In-Memory Mode (인메모리 모드)`
- 데이터를 메모리에 저장하고 처리하는 모드입니다. 데이터베이스가 메모리 내에서 동작하기 때문에 매우 빠른 속도로 작업을 수행할 수 있습니다. 
- 이 모드는 주로 `테스트`나 프로토타이핑 시에 사용되며, 데이터는 임시로 유지되며 데이터베이스를 종료하면 데이터가 사라집니다. 
- `jdbc:h2:mem:`과 같이 URL을 통해 In-Memory 모드를 사용할 수 있습니다.

#### 2. `Embedded Mode (내장 모드)`
- 애플리케이션 내부에서 H2 데이터베이스를 내장하여 사용하는 모드입니다. 애플리케이션과 데이터베이스가 같은 JVM에서 동작하기 때문에 별도의 데이터베이스 서버가 필요하지 않습니다.
- 이 모드 역시 주로 테스트 목적으로 사용되며, 데이터는 애플리케이션 실행 중에 유지됩니다. 
- `jdbc:h2:~/test`와 같이 URL을 통해 Embedded 모드를 사용할 수 있습니다.

#### 3. `Server Mode`
- H2 데이터베이스 서버를 별도로 실행하고, 다른 애플리케이션들이 네트워크를 통해 해당 서버에 연결하여 사용하는 모드입니다. 
- 여러 클라이언트가 동시에 접근할 수 있고, 데이터베이스는 영구적으로 유지됩니다. 서버 모드는 실제 운영 환경에서 사용되기도 하며, 데이터베이스 관리 및 공유가 필요한 경우 유용합니다. 
- `jdbc:h2:tcp://localhost/~/test`와 같이 URL을 통해 Server 모드를 사용할 수 있습니다.


---
### PART 3. In-Memory H2 데이터 베이스 환경 구축하기

#### 1. build.gradle에 h2 의존성 추가하기
```gradle
 //jpa
 implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

 //test
 testImplementation 'org.springframework.boot:spring-boot-starter-test'
 testRuntimeOnly 'com.h2database:h2'
```


#### 2. application-test.yml 파일 작성하여 h2 환경 설정하기

```yml
spring:
  datasource :
    url: jdbc:h2:mem:test
    driverClassName: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect

```

`spring.datasource.url : jdbc:h2:mem:test;`

yml 파일 중에서 내장형 H2 데이터 베이스를 연결할 때에는 위의 작성한 부분이 가장 핵심적인 부분입니다.

위에서 언급했듯이, `jdbc:h2:mem:`과 같이 URL을 통해 In-Memory 모드를 사용할 수 있습니다.

![Alt text](/assets/img/2023-08-28/image.png)
- 해당 application-test.yml 파일은 test 폴더 > resource에 위치하도록 하였습니다.

#### 3. `@TestPropertySource`으로 테스트 속성 자료의 위치 선정하기

`@TestPropertySource(locations = "classpath:application-test.yml")` 어노테이션을 사용하여, 테스트 시 property source의 위치를 설정해줄 수 있습니다.

#### @TestPropertySource 사용 예시 코드
```java
@DataJpaTest
@TestPropertySource(locations = "classpath:application-test.yml")
public class ScrapRepositoryTest {
    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        User user = User.builder()
                .name("koko")
                .provider(Provider.GOOGLE)
                .role(Role.USER)
                .email("1234@naver.com")
                .profileUrl("https://www.naver.com")
                .build();

        userRepository.save(user);
    }

    @Test
    void IfAUserExistsThenFindByEmailAndDeletedDateIsNullReturnsSuccess() {
        // given
        //when
        Optional<User> result = userRepository.findByEmailAndDeletedDateIsNull("1234@naver.com");

        // then
        assertThat(result.isPresent()).isTrue();
    }
}
```

출처 : <a href="https://kukim.tistory.com/105"> https://kukim.tistory.com/105 </a>

---

### PART 4. 테스트 코드 H2 사용시 발생한 문제점과 해결책

drop table if exists user CASCADE " via JDBC Statement에 대한 에러가 발생하였습니다.

![Alt text](/assets/img/2023-08-28/image-1.png)

```log
org.hibernate.tool.schema.spi.CommandAcceptanceException: Error executing DDL "
    drop table if exists user CASCADE " via JDBC Statement
	at org.hibernate.tool.schema.internal.exec.GenerationTargetToDatabase.accept(GenerationTargetToDatabase.java:67) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.applySqlString(SchemaDropperImpl.java:387) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.applySqlStrings(SchemaDropperImpl.java:371) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.dropFromMetadata(SchemaDropperImpl.java:246) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.performDrop(SchemaDropperImpl.java:156) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.doDrop(SchemaDropperImpl.java:128) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.internal.SchemaDropperImpl.doDrop(SchemaDropperImpl.java:114) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.spi.SchemaManagementToolCoordinator.performDatabaseAction(SchemaManagementToolCoordinator.java:157) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.tool.schema.spi.SchemaManagementToolCoordinator.process(SchemaManagementToolCoordinator.java:85) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.internal.SessionFactoryImpl.<init>(SessionFactoryImpl.java:335) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.boot.internal.SessionFactoryBuilderImpl.build(SessionFactoryBuilderImpl.java:471) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
	at org.hibernate.jpa.boot.internal.EntityManagerFactoryBuilderImpl.build(EntityManagerFactoryBuilderImpl.java:1498) ~[hibernate-core-5.6.15.Final.jar:5.6.15.Final]
```

- 현재 User에 대한 아래와 같이 Entity 테이블을 사용하고 있습니다.
이때, 해당 테이블 명은 user입니다. 
- MySQL에서는 user가 예약어가 아니었지만, `H2 Database가 2.1.212 버전에서 user 키워드가 예약어로 지정`되었습니다. 따라서, 로컬 MySQL에서는 에러가 발생하지 않았지만, H2 데이터 베이스 사용시에는 User 테이블을 생성하지 못하는 에러가 발생하였습니다.
- 따라서, 아래의 코드와 같이 `@Table(name = "users")`를 `추가`하여 예약어인 user를 사용하지 않고 테이블을 생성하여 관련 문제를 해결하였습니다.


```java
@Getter
@NoArgsConstructor
@Entity
@ToString(exclude = "scrapList")
@Table(name = "users")
public class User extends BaseTimeEntity implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "user_id")
    private Long id;

    @OneToMany(mappedBy = "user")
    private List<Scrap> scrapList = new ArrayList<>();

    @Column(length = 100, nullable = false)
    private String name;

    @Column(length = 320, nullable = false)
    private String email;

    @Column(length = 2083, nullable = false)
    private String profileUrl;

    @Column(nullable = false)
    private Provider provider;

    private String uuid;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    @Builder
    public User(String name, String email, String profileUrl, Provider provider, Role role) {
        this.name = name;
        this.email = email;
        this.profileUrl = profileUrl;
        this.provider = provider;
        this.role = role;
    }

    public User update(String name, String profileUrl) {
        this.name = name;
        this.profileUrl = profileUrl;
        return this;
    }

    public String getRoleKey() {
        return this.role.getKey();
    }
}
```

![Alt text](/assets/img/2023-08-28/image-2.png)