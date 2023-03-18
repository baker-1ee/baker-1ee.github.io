# EventListener

Spring 에서의 Event listener 는 무엇이고, 사용하면 어떠한 이점이 있는지 간단한 코드를 작성해 비교해보면서 정리해보자.

우선 spring event listener 에서 말하는 event 란 어떤 이벤트가 발생했을 때, 리스너에게 전달이 필요한 데이터를 저장할 클래스 정도로 보면 될 것 같다.

그리고, publisher 는 event 를 발행하는자, listener 는 event 가 발행되면 수신하여 처리하는자 정도로 정리하자.

event listener 가 적용되지 않은 코드를 먼저 살펴보자.



## 회원가입 후 여러가지 처리가 필요한 서비스

상황을 가정해보자. 사용자가 회원가입을 하면 신규 유저에게 웰컴메일을 보내고 시스템 내부적으로 유저 관련 현황을 재집계 하는 서비스가 있다고 하자.

코드는 아래와 같이 작성했다.

##### User

```java
@Data
@Builder
public class User {
    private Long id;
    private String email;
    private String password;

    public static User newInstance(String email, String password) {
        return User.builder()
                .email(email)
                .password(password)
                .build();
    }
}
```

간단하게 사번, 이메일, 비밀번호만 저장할 유저 클래스



##### UserRepository

```java
public interface UserRepository {

    User save(User user);

    Optional<User> findById(Long id);

}
```

유저 인스턴스를 저장하고 조회할 UserRepository 인터페이스



##### UserMemoryRepository

```java
@Repository
public class UserMemoryRepository implements UserRepository {

    private static final Map<Long, User> store = new ConcurrentHashMap<>();
    private static final AtomicLong sequence = new AtomicLong();

    @Override
    public User save(User user) {
        if(user.getId() == null) {
            user.setId(sequence.incrementAndGet());
        }
        store.put(user.getId(), user);
        System.out.println(user.getId() + " 사번으로 회원 가입 되었습니다.");
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

유저를 Map 에 저장하여 관리할 UserRepository 구현체 클래스



##### SignUpService

```java
@Service
@RequiredArgsConstructor
public class BeforeSignUpService {

    private final UserRepository userRepository;
    private final SendEmailService sendEmailService;
    private final AggregateUserService aggregateUserService;

    public void signUp(String email, String password) {
        User user = User.newInstance(email, password);
        userRepository.save(user);
        sendEmailService.sendEmail(user);
        aggregateUserService.reAggregate(user);
    }
}
```

회원가입 시 신규 유저 인스턴스를 생성하여 저장하고 웰컴 메일 전송 및 유저 현황 재집계 처리를 하는 회원가입 서비스



##### SendEmailService

```java
@Service
public class SendEmailService {
    public void sendEmail(User user) {
        System.out.println(user.getEmail() + " 에게 email 을 전송하였습니다.");
    }
}
```

메일 전송 서비스



##### AggregateUserService

```java
@Service
public class AggregateUserService {
    public void reAggregate(User user) {
        System.out.println(user.getEmail() + " 회원 추가하여 유저 현황 재집계 완료하였습니다.");
    }
}
```

유저 현황 집계 서비스



위의 샘플 코드 중 SignUpService 를 보면, signUp (회원가입) 메서드에서 신규 회원 생성 및 저장 후 부가처리 (메일 전송, 유저 재집계) 를 하고 있는데, 어찌 생각해보면 메일전송 서비스와 유저 현황 집계 서비스를 회원가입 서비스가 의존하고 있는게 적합할까? 라는 생각이 들 수 있다.

회원 가입 후 웰컴 메일이 아니라 카카오톡을 보내도록 요구사항이 수정될 수도 있고, 회원 가입후 할인 쿠폰을 적용하는 요구사항이 추가 될 수도 있을텐데, 그때마다 회원가입 서비스에 다른 서비스들을 주입받아야 할 것이다.



관심사의 분리를 목적으로 이벤트 리스너를 적용해보자.



## Event Listener 적용



##### SignUpEvent

```java
@Data
@Builder
public class SignUpEvent {
    User user;

    public static SignUpEvent from(User user) {
        return SignUpEvent.builder()
                .user(user)
                .build();
    }
}	
```

회원가입 이벤트 클래스



##### SignUpEventPublisher

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

 ApplicationEventPublisher 를 주입받아서 publishEvent 메서드를 호출해주면 된다.



##### SignUpEventListener

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

@EventListener 어노테이션만 작성해주면 SignUpEvent 가 publish 되면 메서드가 실행된다.
@Order 어노테이션으로는 실행순서를 설정해주었다. 순서가 상관없으면 작성 안하면 된다.



##### SignUpService

```java
@Service
@RequiredArgsConstructor
public class SignUpService {

    private final UserRepository userRepository;
    private final SignUpEventPublisher signUpEventPublisher;

    public void signUp(String email, String password) {
        User user = User.newInstance(email, password);
        userRepository.save(user);
        signUpEventPublisher.publish(user);
    }
}
```

신규 유저를 생성 및 저장후 signUpEventPublisher 의 publish 만 호출해주고 signUp 메서드는 종료된다.
SignUpService 는 더 이상 회원가입 후처리는 관심사가 아니다.
회원가입 후처리는 SignUpEventListener 에서 주관하며, 후처리가 추가되거나 수정될 때는 해당 컴포넌트만 수정하면 된다.



## 맺음말

- Event publishing 은 다른 많은 것들과 마찬가지로 ApplicationContext 에서 제공하는 기능 중 하나이다.
- Spring Framework 4.2 이전 버전까지는 이벤트 클래스는 ApplicationEvent 를 확장해야만 했는데, Spring Framework 4.2 부터는 확장할 필요가 없어졌다.

- Event 처리는 기본적으로 동기식으로 동작한다. 비동기로 처리할 수도 있다.
- @transactionaleventlistener 은 다음 시간에 정리해보자.



## Reference

[Spring Events](https://www.baeldung.com/spring-events)







