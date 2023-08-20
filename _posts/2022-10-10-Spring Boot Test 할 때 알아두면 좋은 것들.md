[스프링부트 공식 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing) 를 참고하여 spring boot test 할 때, 알아두면 좋은 것들을 정리해보자.



## Mocking and Spying

테스트를 실행할 때, ApplicationContext 내의 특정 컴포넌트들을 조작해야 할 필요가 있다.

개발 중에 사용할 수 없는 일부 원격 서비스가 있는 경우 테스트를 위해 Mocking 과 Spying 은 유용한 도구가 될 수 있다.



## @SpringBootTest

- SpringApplication 을 사용하여 ApplicationContext를 로드한다. 
- 기본적으로 @SpringBootTest 만으로는 서버를 실행하지 않기 때문에, web environment test 를 하려면 추가 속성이 필요하다.
- 어플리케이션의 설정과 모든 빈을 로드하기 때문에 시간이 오래 걸리므로 단위 테스트 보다는 통합 테스트에 적합하다.
  - JUnit5부터는 @RunWith, @ExtendWith 등을 추가해줄 필요가 없다.



## @AutoConfigureMockMvc

- Mock 테스트 시 필요한 의존성을 제공한다.

  ```java
  @Autowired
  MockMvc mvc;
  ```

- @WebMvcTest 가 아닌 @SpringBootTest 에서도 Mock 테스트를 가능하게 해주는 역할을 한다.



## @WebMvcTest

- 웹 계층에만 집중하고 전체 ApplicationContext 를 시작하지 않고자 할 때 사용한다.
- Controller 가 예상대로 동작하는지 테스트하기 위해 사용한다.
- WebApplication 과 관련된 빈들만 스캔하여 등록하기 때문에 @SpringBootTest 보다 빠르다.



## Web endpoints test with Spring MVC

> @SpringBootTest + @AutoConfigureMockMvc 의 조합으로 Controller 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
public class AdminControllerTest {

    @InjectMocks
    private AdminController adminController;

    @Mock
    private AdminService adminService;

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getAdminTest() {
        // given
        String name = "이한슬";
        String email = "sky@naver.com";

        AdminResponse adminResponse = AdminResponse.builder().name(name).email(email).build();
        BDDMockito.given(adminService.getAdmin()).willReturn(adminResponse);

        // when
        ResponseEntity<AdminResponse> responseEntity = adminController.getAdmin();

        // then
        assertThat(responseEntity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(Objects.requireNonNull(responseEntity.getBody()).getName()).isEqualTo(name);
        assertThat(Objects.requireNonNull(responseEntity.getBody()).getEmail()).isEqualTo(email);
    }

    @Test
    void loggingInterceptor_가_정상_등록되어있다면_response_body_를_콘솔에_출력해준다() throws Exception {
        // given
        String name = "이한슬";
        String email = "sky@naver.com";

        AdminResponse adminResponse = AdminResponse.builder().name(name).email(email).build();
        BDDMockito.given(adminService.getAdmin()).willReturn(adminResponse);

        // when
        ResultActions resultActions = mockMvc.perform(get("/admin")
                .contentType(MediaType.APPLICATION_JSON));

        // then
        resultActions.andExpect(status().isOk()).andDo(print());
    }

}
```



## web endpoints test with Spring WebFlux

> @SpringBootTest + @AutoConfigureWebTestClient 의 조합으로 Controller 테스트

```java
@SpringBootTest
@AutoConfigureWebTestClient
class MyMockWebTestClientTests {

    @Test
    void exampleTest(@Autowired WebTestClient webClient) {
        webClient
            .get().uri("/")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class).isEqualTo("Hello World");
    }

}
```



## SpringBootTest 의 Test double

#### Test Double

> 테스트 할 때, 실제 객체를 대신 모의 객체로 테스트 하는 방법으로 Java 진영에서는 대표적으로 Mockito 가 있다.



#### Mockito

- @Mock
  - mock 객체 생성
- @MockBean
  - mock 객체 생성 및 스프링 컨텍스트에 등록
  - @Autowired 로 의존성 주입
- @Spy
  - 실제 인스턴스를 사용해서 Mocking 할 수 있음
- @SpyBean
  - 스프링 컨테이너에 Bean 으로 등록된 객체에 대해 Spy 를 생성
  - 실제 구현된 객체를 감싸는 프록시 객체 형태이므로 스프링 컨텍스트에 실제 구현체가 등록되어 있어야 한다.
- @InjectMocks
  - @Mock 이 붙은 객체를 @InjectMocks 이 붙은 객체에 주입 시킨다.

