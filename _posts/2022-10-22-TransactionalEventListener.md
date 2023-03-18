[지난 시간에 EventListener 적용](https://zeemoong.github.io/EventListener/) 까지는 진행해봤는데, 이번 시간에는 TransactionalEventListener 와의 차이를 비교해보자.

SignUpService 의 signUp 메소드에서는 신규 유저 엔터티를 생성하여 저장하고, signUpEvent 를 발행 후 끝난다.
signUpEventListener 에서는 SendEmailService 에서 메일을 전송하고, AggregateUserService 에서 유저 현황을 재집계 처리를 한다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class SignUpService {

    private final UserRepository userRepository;
    private final SignUpEventPublisher signUpEventPublisher;

    @Transactional
    public void signUp(String email, String password) {
        User user = User.newInstance(email, password);
        userRepository.save(user);
        signUpEventPublisher.publish(user);
        System.out.println("user = " + user);
    }
}
```



```java
@Component
@RequiredArgsConstructor
public class SignUpEventPublisher {
    private final ApplicationEventPublisher applicationEventPublisher;

    public void publish(User user) {
        SignUpEvent event = SignUpEvent.from(user);
        applicationEventPublisher.publishEvent(event);
    }
}
```



```java
@Component
@RequiredArgsConstructor
public class SignUpEventListener {

    private final SendEmailService sendEmailService;
    private final AggregateUserService aggregateUserService;

    @EventListener
    @Order(1)
    public void sendEmail(SignUpEvent event) {
        sendEmailService.sendEmail(event.getUser());
    }

    @EventListener
    @Order(2)
    public void reAggregate(SignUpEvent event) {
        aggregateUserService.reAggregate(event.getUser());
    }
}
```



```java
@Service
public class SendEmailService {
    public void sendEmail(User user) {
        System.out.println(user.getEmail() + " 에게 email 을 전송하였습니다.");
    }
}
```



```java
@Service
public class AggregateUserService {
    public void reAggregate(User user) {
        System.out.println(user.getEmail() + " 회원 추가하여 유저 현황 재집계 완료하였습니다.");
    }
}
```



아래와 같이 signUpTest 메소드가 실행된다면 콘솔에는 어떤 결과가 출력될까?

```java
@SpringBootTest
class SignUpServiceTest {

    @Autowired
    private SignUpService signUpService;

    @Test
    void signUpTest() {
        signUpService.signUp("sky@naver.com", "pass1234");
    }

}
```



event listener 가 기본적으로 동기 방식으로 처리하기 때문에 아래와 같이 로직 순서대로 콘솔 출력이 된다.

```text
sky@naver.com 에게 email 을 전송하였습니다.
sky@naver.com 회원 추가하여 유저 현황 재집계 완료하였습니다.
user = User(id=1, email=sky@naver.com, password=pass1234)
```



그렇다면, signUp 메소드에서 RuntimeException 이 발생하는 경우라면 어떻게 될까?

```java
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class SignUpService {

    private final UserRepository userRepository;
    private final SignUpEventPublisher signUpEventPublisher;

    @Transactional
    public void signUp(String email, String password) {
        User user = User.newInstance(email, password);
        userRepository.save(user);
        signUpEventPublisher.publish(user);
        System.out.println("user = " + user);
        throw new RuntimeException("roll back");
    }
}
```



event listener 로직들이 모두 실행 된 뒤 exception 이 발생하여 User 엔터티는 저장되지 않고 롤백 될 것이다.

```text
sky@naver.com 에게 email 을 전송하였습니다.
sky@naver.com 회원 추가하여 유저 현황 재집계 완료하였습니다.
user = User(id=1, email=sky@naver.com, password=pass1234)

java.lang.RuntimeException: roll back
```



만약, signUp 메소드가 성공적으로 커밋 된 뒤에 후처리 로직을 실행해야만 하는 상황이라면 어떻게 해야 할까?

@EventListener 대신 @TransactionalEventListener 를 사용하면 된다.

```java
@Component
@RequiredArgsConstructor
public class SignUpEventListener {

    private final SendEmailService sendEmailService;
    private final AggregateUserService aggregateUserService;

    @TransactionalEventListener
    @Order(1)
    public void sendEmail(SignUpEvent event) {
        sendEmailService.sendEmail(event.getUser());
    }

    @TransactionalEventListener
    @Order(2)
    public void reAggregate(SignUpEvent event) {
        aggregateUserService.reAggregate(event.getUser());
    }
}
```



결과는 예상대로 아래와 같이 콘솔 출력 될 것이다.

```text
user = User(id=1, email=sky@naver.com, password=pass1234)

java.lang.RuntimeException: roll back
```



만약, Exception 이 발생하지 않는 정상적인 케이스라면?
아래와 같이 signUp 메소드가 정상적으로 종료된 후 이벤트 리스너 등록 메소드가 실행된다.

```text
user = User(id=1, email=sky@naver.com, password=pass1234)
sky@naver.com 에게 email 을 전송하였습니다.
sky@naver.com 회원 추가하여 유저 현황 재집계 완료하였습니다.
```



### @TransactionalEventListener 옵션

`@TransactionalEventListener`을 이용하면 트랜잭션의 어떤 타이밍에 이벤트를 발생시킬 지 정할 수 있습니다. 옵션을 사용하는 방법은 `TransactionPhase`을 이용하는 것이며 아래와 같은 옵션을 사용할 수 있습니다.

- **AFTER_COMMIT (기본값)** - 트랜잭션이 성공적으로 마무리(commit)됬을 때 이벤트 실행
- AFTER_ROLLBACK – 트랜잭션이 rollback 됬을 때 이벤트 실행
- AFTER_COMPLETION – 트랜잭션이 마무리 됬을 때(commit or rollback) 이벤트 실행
- BEFORE_COMMIT - 트랜잭션의 커밋 전에 이벤트 실행



[git 링크](https://github.com/zeemoong/spring/tree/feature/101)



