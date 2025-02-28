# 어플리케이션 테스트 

테스트는 크게 단위 테스트와 통합 테스트로 구분할 수 있다.<br>
어디까지 테스트 하는 것이 단위 테스트에 해당하는지, 통합 테스트는 어느 영역까지 진행해야하는지 모호하다.<br>
이 부분에 관해 많은 포스팅, 레퍼런스, 영상을 보았는데, 이 부분에 대해 적절한 답변을 찾은듯하여 정리하여 공유한다.<br>
<br>

- 단위 테스트에서는 Spring context를 지양하며, Java 수준에서 동작할 수 있도록 한다.
- 통합 테스트에서는 Spring context를 활용하고, 경우에 따라 MockBean을 활용할 수 있다.
    - kafka, email, 외부 MSA 서비스에 대해 MockBean 활용

<br>

## API 테스트는 단위? 통합?
API 테스트는 MVC의 모든 레이어를 관통하는 테스트이다. 그럼 API 테스팅은 단위 테스트인가? 전체를 아우르니 통합테스트인가?<br>
처음 설명했던것처럼 Spring context를 사용하지 않는 것이라고 하였으므로 단위 테스트라고 하기 어렵다.<br>
API 테스트(대부분 Controller에 대한 검증)는 통합 테스트의 격으로 관련 레이어 및 모듈에 대해 수행할 수 있고, Controller에 정의된 메소드만을 위해 수행할 수 있다고 생각된다.<br>
API 테스팅은 단위와 통합 사이에 존재하는, 다소 모호하게 위치한 테스트라고 할 수 있다.<br>
UnitTest, IntegrationTest 또는 그 이외의 별도의 디렉터리에서 관리할지는 회사의 컨벤션을 따르는게 옳다고 할 수 있다.<br><br>



<br>

---

<br>


## 단위 테스트 (비즈니스 로직 검증)

단위 테스트는 어플리케이션의 개별적인 컴포넌트 또는 유닛에 대해 독립적으로 테스트를 수행하는 것을 말한다.<br>
보통 클래스와 각 메소드에 대해 개별적으로 테스트 코드를 작성한다.<br><br>

> `단위 테스트`를 위해서라면 Spring, Spring Boot를 사용하지 않아도 된다.<br>
    스프링을 사용하면 테스트를 위해 스프링을 띄우는데 많은 리소스가 사용된다.<br> 
    가령, UserService의 메소드들에 대해 테스트를 수행할때 Spring을 사용해서 context를 띄우게 되면, UserService 테스트와 무관한 Bean도 등록하기 위해 동작을 수행해야 한다. ie. BankingService or Back-office things etc

<br>

```Java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class RegisterUseCaseTest {

  @Autowired
  private RegisterUseCase registerUseCase;

  @Test
  void savedUserHasRegistrationDate() {
    User user = new User("zaphod", "zaphod@mail.com");
    User savedUser = registerUseCase.registerUser(user);
    assertThat(savedUser.getRegistrationDate()).isNotNull();
  }

}
```

다음과 같이 테스트 클래스를 작성하게되면 @SpringBootTest 애노테이션으로 인해 어플리케이션의 전체 context를 설정한다.<br>
테스트를 위해 context 전체를 구성해야하기에 리소스 낭비가 발생하게 되는 것이다.<br>
<br>

그럼 스프링 안쓰고 어떻게 테스트하나?<br>
<br>
단위 테스트의 경우, 테스트에 사용되는 케이스(UseCase instance)를 주입시켜서 테스트를 수행하도록 한다.<br><br>

### 테스트에 사용될 Bean 생성하기

```Java
@Service
public class RegisterUseCase {
    @Autowired
    private UserRepository userRepository;

    public User registerUser(User user) {
        return userRepository.save(user);
    }
}
```
다음과 같은 유저 등록을 위한 케이스를 생성한다고 하자.<br>
위의 use-case 클래스는 스프링이 없다면 UserRepository 인스턴스를 주입 받지 못하기때문에 사용할 수 없다. 따라서 위의 필드주입 방식으로는 사용할 수 없다.<br>


```Java
@Service
@RequiredArgsConstructor
public class RegisterUseCase {

    private final UserRepository userRepository;

    public User registerUser(User user) {
        user.setRegistrationDate(LocalDateTime.now());
        return userRepository.save(user);
    }

} 


```

이제 위의 생성자 주입방식을 통해서 use-case에 대해 필요한 인스턴스(repository)를 주입할 수 있다.

```Java
class RegisterUseCaseTest {

    private UserRepository userRepository = ...;
    private RegisterUseCase registerUseCase;

    @BeforeEach
    void initUseCase() {
        registerUseCase = new RegisterUseCase(userRepository);
    }

    @Test
    void savedUserHasRegistrationDate() {
        User user = new User("john", "john@gamil.com");
        User savedUser = registerUseCase.registerUser(user);
        assertThat(savedUser.getRegistrationDate()).isNotNull();
    }
}
```

이제 UseCase 클래스가 사용되는 Test코드를 확인할 수 있다.<br>
UseCase 내부에 Repository 필드가 존재한다. Test code <-> Use Case (Repisotry)의 형태로 구성되는 것이다.<br>
즉, 직접 Repository를 사용하는 것이 아니라, Use Case 내부의 설정된 userRepository를 사용한다.<br>


<br>

---

<br>


### Mockito를 활용해 Mock Dependency 사용하기

- Mockito.mock() 사용하기

다음의 예시에서 mock 객체를 생성해서 조작하는 과정을 담았다.

```Java
// userRepository에 대해 mock wrap up
private UserRepository userRepository = Mockito.mock(UserRepository.class);
// User 객체 생성
User user = new User("john", "john@gmail.com");
// stubbing for mocking 
when(userRepository.save(any(User.class))).then(returnsFirstArg());
// mock객체는 stubbing 된것에 따라 같은 동작을 수행
User savedUser = registerUseCase.registerUser(user);
```
mock 객체는 앞서 언급한것처럼 기본적으로 아무런 논리적 동작을 수행하지 않으며, 리턴값으로 null, 빈 collection, 기본값 등을 리턴한다.
해당 mock 객체의 메소드에 대해 override해서 동작에 대한 수행을 셋업하는데, 이를 `stub` 이라고 한다.
stubbing 이후에 다시 stubbing 하지 않는다면 계속 같은 값을 리턴한다. 
<br>
위의 예시에서 userRepository는 Mockito의 mock()을 통해 생성된 mockup 객체이다.<br>
이후 when(), then()을 통해서 특정 메소드에 대해 파라미터를 정의하고, 해당 케이스에 어떤 값을 반환할지 정의했다.<br>
마지막 줄에서 앞서 stubbing 된것에 따라 user 객체를 반환한다.<br>

<br>
다음은 stubbing하는 예시 코드를 공식 문서에서 발췌했다. stubbing관련은 공식 레퍼런스를 참조.

```Java
 LinkedList mockedList = mock(LinkedList.class);

//stubbing using built-in anyInt() argument matcher
 when(mockedList.get(anyInt())).thenReturn("element");

 //stubbing using custom matcher (let's say isValid() returns your own matcher implementation):
 when(mockedList.contains(argThat(isValid()))).thenReturn("element");

 //following prints "element"
 System.out.println(mockedList.get(999));

 //you can also verify using an argument matcher
 verify(mockedList).get(anyInt());

 //argument matchers can also be written as Java 8 Lambdas
 verify(mockedList).add(argThat(someString -> someString.length() > 5));
```


- Mockito 애노테이션 활용
앞의 예시에서는 Mockito의 메소드를 직접 사용해서 mock 객체를 생성해서 조작했다.<br>
JUnit에 MockitoExtension을 확장하여 @Mock 애노테이션을 활용하면 훨씬 편하게 테스트 코드를 작성할 수 있다.<br>

```Java
@ExtendWith(MockitoExtension.class)
class RegisterUseCaseTest {


  // private UserRepository userRepository = Mockito.mock(UserRepository.class);
  @Mock
  private UserRepository userRepository;

  // private RegisterUseCase registerUseCase;
  @InjectMocks
  private RegisterUseCase registerUseCase;


  @BeforeEach
  void initUseCase() {
    registerUseCase = new RegisterUseCase(userRepository);
  }

  @Test
  void savedUserHasRegistrationDate() {
    // ...
  }

}
```

`@MockitoExtension`은 Mockito가 @Mock 애노테이션을 탐지해서 해당 필드에 대해 mock 객체를 주입시킨다.<br>
`@InjectMocks` 애노테이션은 앞서 직접 UseCase클래스를 생성했던것을 대신 수행해준다.<br>
<br>

### custom Assert 사용하기

```Java
class UserAssert extends AbstractAssert<UserAssert, User> {

  UserAssert(User user) {
    super(user, UserAssert.class);
  }

  static UserAssert assertThat(User actual) {
    return new UserAssert(actual);
  }

  UserAssert hasRegistrationDate() {
    isNotNull();
    if (actual.getRegistrationDate() == null) {
      failWithMessage(
        "Expected user to have a registration date, but it was null"
      );
    }
    return this;
  }
}
```
다음과 같이 `AbstractAssert abstract class`를 상속해서 `Custom Assert`를 활용할 수 있다.<br>
이 부분에 대해서는 개인적으로 의견이 분분하다. 참조한 글에서는 위의 Custom Assert를 통해 테스트 코드에 대한 가독성을 챙길 수 있다고 소개한다.<br>
하지만, 많은 테스트코드에 대해 직접 메소드를 구현하면 오히려 작업이 더 많아지는게 아닌지 생각이 든다.<br>
한편, 동의하는 부분은 위의 예시에서 처럼 User에 대한 다양한 테스트를 반복해서 해야한다면 시간이 걸리더라도 괜찮은 방법이라고 느껴진다.<br><br>

```Java
// Junit만을 사용한 테스트코드
assertThat(savedUser.getRegistrationDate()).isNotNull();

// Custom Assert를 활용한 테스트코드
assertThat(savedUser).hasRegistrationDate();
```





<br>

---

<br>



## 통합 테스트 (Boot Test, Web MVC Test, DataJpaTest)

통합 테스트는 애플리케이션 시스템을 구성하는 이종의 컴포넌트에 대해 그룹 단위로 동작을 검증하여 테스트를 수행하는 것을 의미한다.<br><br>


### Web MVC Test
```Java
@RestController
@RequiredArgsConstructor
class RegisterRestController {
  private final RegisterUseCase registerUseCase;

  @PostMapping("/forums/{forumId}/register")
  UserResource register(
          @PathVariable("forumId") Long forumId,
          @Valid @RequestBody UserResource userResource,
          @RequestParam("sendWelcomeMail") boolean sendWelcomeMail) {

    User user = new User(
            userResource.getName(),
            userResource.getEmail());
    Long userId = registerUseCase.registerUser(user, sendWelcomeMail);

    return new UserResource(
            userId,
            user.getName(),
            user.getEmail());
  }

}
```
위와 같은 Controller가 정의되어 있다면, 요청에 대해 응답을 리턴하는 과정을 다음의 순서로 설명할 수 있다.<br>

1. 정의된 URL, HTTP메소드, 컨텐츠에 따라 요청을 받는다.
2. Controller는 HTTP 요청을 Java 객체의 형태로 변환(Deserialize)한다.
3. 주어진 입력에 대해 검증을 수행하며, 정상적인 입력이 아닌 경우 예외를 처리한다.
4. Service 레이어의 비즈니스 로직을 수행한다. (unit test)
5. 비즈니스 로직 수행의 결과물을 HTTP 응답으로 보내기 위한 변환(Serialize)을 수행한다.
6. 예외가 발생한 경우, 에러 메세지와 HTTP 상태를 응답으로 전송한다.

위의 일련의 과정에서 알 수 있듯, Controller는 많은 책임을 지니고 있으며 다른 레이어와 연관관계를 맺고 있다.<br>
위의 모든 책임에 대한 동작 검증을 테스트하고, 연관있는 레이어에 대해서도 테스트를 수행해야한다면 Controller에 대한 테스트가 너무도 많아지고 복잡해진다.<br><br>

따라서 Controller에 대한 테스트는 위의 언급한 책임에 대해 나눠서 테스트코드를 설계하도록 한다.<br>
Controller에 대해 수행할 수 있는것은 4번의 비즈니스 로직의 수행에 대해서만 가능하며, 이외의 작업은 HTTP 레이어에 대한 설정이 필요하다.<br>
특정 URL에 대해 Controller가 요청을 받는지, JSON형태로 변환이 잘 일어나는지 확인하기 위해 @WebMvcTest 어노테이션을 사용한다.<br><br>

```Java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = RegisterRestController.class)
class RegisterRestControllerTest {
  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @MockBean
  private RegisterUseCase registerUseCase;

  @Test
  void whenValidInput_thenReturns200() throws Exception {
    mockMvc.perform(...);
  }

}
```

> Spring Boot 2.1이후부터는 `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`에 **SpringExtension**이 포함되므로 직접 애너테이션을 명시하지않아도 된다.

Spring Boot는 @WebMvcTest 어노테이션을 통해 Controller에 필요한 application context를 제공한다.<br>
위의 예시코드에서 @Autowire가 붙은 필드에 대해 자동으로 빈을 주입해준다.<br>
또한, 앞서 언급한 단위 테스트의 Mockito의 @MockBean을 통해 비즈니스 로직의 동작에 대해 mock-up을 설정한다.<br>

- 1 -> HTTP 요청에 대한 검증
```Java
mockMvc.perform( post("/some/url")
    .contentType("application/json")
    .andExpect(status().isOk()));
```
MockMvc의 메소드 체이닝을 통해 HTTP 요청을 생성하고 테스트를 수행할 수 있다.<br>

- 2 -> 요청의 파라미터에 대한 검증 (Deserialization)
HTTP 요청을 통해 본문(@RequestBody), URL 경로(@PathVariable), 또는 요청 파라미터(@RequestParam)을 받을 수 있다.<br>
이에 대한 검증을 위해 MockMvc에서 다음과 같은 방식으로 설정하여 테스트를 수행한다.<br>

```Java
@Test
void whenValidInput_thenReturns200() throws Exception {
  UserResource user = new UserResource("Zaphod", "zaphod@galaxy.net");
  
   mockMvc.perform(post("/forums/{forumId}/register", 42L)
        .contentType("application/json")
        .param("sendWelcomeMail", "true")
        .content(objectMapper.writeValueAsString(user)))
        .andExpect(status().isOk());
}
```
content()메소드에서 ObjectMapper를 통해 생성한 User 객체에 대해 JSON형식으로 직렬화시켜준다.<br>
<br>

- 3 -> input에 대한 검증 (null check)
요청의 파라미터에 대한 Null 검증은 Domain 클래스의 @NotNull 애노테이션을 통해서 수행된다.<br>
이에 대한 예시는 다음의 코드와 같다.<br>

```Java
@Value
public class User {

  @NotNull
  private final String name;

  @NotNull
  private final String email;
  
}

@Test
void whenNullValue_thenReturns400() throws Exception {
  User user = new User(null, "zaphod@galaxy.net");
  
  mockMvc.perform(post("/forums/{forumId}/register", 42L)
      ...
      .content(objectMapper.writeValueAsString(user)))
      .andExpect(status().isBadRequest());
}
```
<br>

- 5 -> output에 대한 검증(serialization)
비즈니스 로직을 수행하고, 그 결과에 해당하는 JSON을 반환해야 한다면 해당 결과 값이 HTTP 응답의 본문에 반영되는지 확인해야한다.<br>
```Java
@Test
void whenValidInput_thenReturnsUserResource() throws Exception {
  MvcResult mvcResult = mockMvc.perform(...) // 응답 결과값을 담는 변수 mvcResult
      ...
      .andReturn();

  UserResource expectedResponseBody = ...;
  String actualResponseBody = mvcResult.getResponse().getContentAsString();
  
  assertThat(actualResponseBody).isEqualToIgnoringWhitespace(
              objectMapper.writeValueAsString(expectedResponseBody));
}
```
위의 테스트코드에서 비즈니스 로직에 대한 결과를 받고 String 타입으로 변환하여 실제 응답 본문과 동일한지 검증한다. <br>

- 6 -> 특정 예외발생 시 상태코드 및 메시지 확인


```Java
@Test
void whenNullValue_thenReturns400AndErrorResult() throws Exception {
  UserResource user = new UserResource(null, "zaphod@galaxy.net");

  MvcResult mvcResult = mockMvc.perform(...)
          .contentType("application/json")
          .param("sendWelcomeMail", "true")
          .content(objectMapper.writeValueAsString(user))
          .andExpect(status().isBadRequest()) // 에러코드 검증
          .andReturn();

  ErrorResult expectedErrorResponse = new ErrorResult("name", "must not be null");
  String actualResponseBody = 
      mvcResult.getResponse().getContentAsString();
  String expectedResponseBody = 
      objectMapper.writeValueAsString(expectedErrorResponse);
  assertThat(actualResponseBody)
      .isEqualToIgnoringWhitespace(expectedResponseBody);
}
```

5번에서의 과정과 비슷하지만, 이 경우에는 예외에 대한 검증이기에 상태코드와 ErrorResult 클래스를 통한 에러 응답(response body)를 검증한다.<br>

### Response Body Matcher as JSON
이전의 과정에서는 JSON에 대한 비교를 하기 위해서 String 타입으로 변환하여 검증을 수행했다.<br>
JSON을 변환하는 과정을 위해서 `Custom Matcher`를 활용하면 훨씬 편리하게 사용할 수 있다.<br>

다음의 Custom Response Body Matcher의 예시코드이다.<br>
```Java
public class ResponseBodyMatchers {
  private ObjectMapper objectMapper = new ObjectMapper();

  public <T> ResultMatcher containsObjectAsJson(
      Object expectedObject, 
      Class<T> targetClass) {
    return mvcResult -> {
      String json = mvcResult.getResponse().getContentAsString();
      T actualObject = objectMapper.readValue(json, targetClass);
      assertThat(actualObject).isEqualToComparingFieldByField(expectedObject);
    };
  }
  
  static ResponseBodyMatchers responseBody(){
    return new ResponseBodyMatchers();
  }
  
}

@Test
void whenValidInput_thenReturnsUserResource_withFluentApi() throws Exception {
  UserResource user = ...;
  UserResource expected = ...;

  mockMvc.perform(...)
      ...
      .andExpect(responseBody().containsObjectAsJson(expected, UserResource.class));
}
```

<br><br>

### DataJpaTest
DB와 연관이 있는 컴포넌트에 대해 테스트를 수행하기 위해서는 Persistence 레이어와 함께 테스트를 진행해야 한다.<br>
따라서 이러한 테스트는 통합테스트로 분류하는게 맞다고 판단했다.<br><br>

@DataJapTest 어노테이션을 통해서 DB에 대한 편리한 설정으로 테스트를 수행할 수 있다.<br>
해당 어노테이션을 사용하면 설정된 DB가 아닌 in-memory 방식으로 대체된다.<br>

```Java
@DataJpaTest
class HibernateTest {
  @Autowired private UserRepository userRepository;
  @Autowired private DataSource dataSource;
  @Autowired private JdbcTemplate jdbcTemplate;
  @Autowired private EntityManager entityManager;

  @Test
  void injectedComponentsAreNotNull() {
    assertThat(dataSource).isNotNull();
    assertThat(jdbcTemplate).isNotNull();
    assertThat(entityManager).isNotNull();
    assertThat(userRepository).isNotNull();
  }

  @Test
  void whenSaved_thenFindsByName() {
    userRepository.save(new UserEntity(
            "Zaphod Beeblebrox",
            "zaphod@galaxy.net"));
    assertThat(userRepository.findByName("Zaphod Beeblebrox")).isNotNull();
  }

  @Test
  void stateIsNotShared1() {
    assertThat(userRepository.findByName("user2")).isNull();
    userRepository.save(new UserEntity("user1", "mail1"));
  }

  @Test
  void stateIsNotShared2() {
    assertThat(userRepository.findByName("user1")).isNull();
    userRepository.save(new UserEntity("user2", "mail2"));
  }

}
``` 
해당 어노테이션 하위에서는 spring context가 공유되며 각 method는 각자의 트랜잭션 내에서 수행되며, 테스트코드가 끝나면 롤백이 지원된다.<br>


### DB를 테스트하기 전에 데이터 준비

- 직접 엔티티 설정
```Java
userRepository.save(new UserEntity(
        "Zaphod Beeblebrox",
        "zaphod@galaxy.net"));
```
앞의 예시와 같이 직접 엔티티를 만들고 Repository에 적재하는 방법이 있다.<br>
하지만, 어플리케이션과 테스트의 복잡도가 높아질수록 사용하기에 부담이 커진다.<br><br>


- data.sql 사용

```sql
INSERT INTO users (id, first_name, last_name, email) VALUES 
                        (1, 'John', 'Doe', 'johndoe@example.com'),
                        (2, 'Jane', 'Doe', 'janedoe@example.com'),
                        (3, 'Bob', 'Smith', 'bobsmith@example.com'),
                        (4, 'Alice', 'Johnson', 'alicejohnson@example.com');
```
insert문을 사전에 작업하여 테스트를 수행하기 전에 DB에 데이터가 적재되도록 할 수 있다. <br>
이를 위해 data.sql 파일은 Spring의 클래스패스 내에 위치해야하며, @Sql() 어노테이션을 통해 스크립트 파일을 명시한다.<br>
<br>

```Java
@DataJpaTest
@Sql(scripts = "classpath:data.sql")
public class UserRepositoryTest {
    @Autowired private UserRepository userRepository;

    // ...
}
```

다음과 같은 경우, 스프링은 해당 테스트 클래스 내부의 테스트를 수행하기 앞서 스크립트 파일을 실행시켜 DB를 채우는 작업을 하게 된다.<br>
<br>
**data.sql**과 같은 스크립트를 사용하게 되면 모든 데이터 준비 스크립트가 한 곳에서 관리하게 된다.<br>
만약, 테스트의 규모가 커지고 경우에 따라 데이터를 달리해야한다면 한 곳의 스크립트에서 관리하는 것은 혼란을 야기한다.<br>
따라서 스크립트를 나누거나 다른 방법 사용을 고려해봐야한다.<br><br>


- Spring DBUnit 사용
DBUnit 라이브러리는 특정 상태의 DB를 제공한다. <br>
해당 라이브러리를 사용하기 위해서는 다음과 같은 의존성(Spring DBUnit, DBUnit)을 설정해야한다.<br>

```yaml
compile('com.github.springtestdbunit:spring-test-dbunit:1.3.0')
compile('org.dbunit:dbunit:2.6.0')
```

그리고 DB의 상태를 정의한 XML파일을 생성한다.<br>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dataset>
    <user
        id="1"
        name="Zaphod Beeblebrox"
        email="zaphod@galaxy.net"
    />
</dataset>
```
해당 xml파일은 테스트 클래스와 동일한 클래스패스에 위치시킨다.<br>
<br>
사전 작업은 모두 끝났으며, 이제 실제 테스트코드에서 이를 활용하기 위한 부분을 확인해보자.<br>

```Java
@DataJpaTest
@TestExecutionListeners({ // enable DBUnit support
        DependencyInjectionTestExecutionListener.class, 
        TransactionDbUnitTestExecutionListener.class    
})
class SpringDbUnitTest {

  @Autowired
  private UserRepository userRepository;

  @Test
  @DatabaseSetup("createUser.xml") // specify xml for DB state
  void whenInitializedByDbUnit_thenFindsByName() {
    UserEntity user = userRepository.findByName("Zaphod Beeblebrox");
    assertThat(user).isNotNull();
  }

}
```


<br>

---

<br>


- reference
    - "Unit Test or Integration Test in Spring Boot", https://stackoverflow.com/questions/54658563/unit-test-or-integration-test-in-spring-boot
    - "Unit Testing with Spring Boot", https://reflectoring.io/unit-testing-spring-boot/
    - "Testing MVC Web Controllers with Spring Boot and @WebMvcTest", https://reflectoring.io/spring-boot-web-controller-test/
    - Mockito Officialy Reference, https://www.javadoc.io/doc/org.mockito/mockito-core/2.23.4/org/mockito/Mockito.html
    - "MockMvc - Spring MVC testing framework introduction: Testing Spring endpoints", https://blog.marcnuri.com/mockmvc-spring-mvc-framework
    - "9. Database Initialization", Spring Reference, https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.data-initialization