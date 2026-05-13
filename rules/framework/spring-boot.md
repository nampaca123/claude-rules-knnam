# Spring Boot

널리 받아들여졌지만 많이 놓치는 것들. Java/Spring 작업 시 반드시 따른다.

## 1. 의존성 주입은 생성자 주입만

필드 주입(`@Autowired private Foo foo;`) 금지. 생성자 주입을 쓰면 `final` 필드 사용 가능, 누락 의존성을 시작 시 탐지, 테스트에서 mocking 쉬움. Spring 4.3부터 단일 생성자에는 `@Autowired` 안 붙여도 동작한다.

생성자 인자가 많아지면(>5) SRP 위반 신호다. 클래스를 분리한다.

## 2. @Transactional은 서비스 계층에

- 컨트롤러에 안 붙인다. (HTTP 진입점이 DB 트랜잭션을 열면 안 된다.)
- Spring Data 리포지토리에 안 붙인다. (메서드 단위로 이미 트랜잭션이 걸려 있다.)
- 비즈니스 유스케이스를 묶는 서비스 메서드에 붙인다.

함정: `@Transactional`은 기본적으로 `RuntimeException`에서만 롤백한다. checked exception을 던지면서 롤백을 원하면 `rollbackFor = Exception.class`. 같은 클래스 내 메서드에서 `@Transactional` 메서드를 호출하면 프록시를 통과하지 않아 트랜잭션이 적용되지 않는다(self-invocation 문제).

## 3. 엔티티를 절대 외부에 노출하지 않는다

`@Entity`를 컨트롤러의 `@RequestBody`나 응답으로 쓰지 않는다. DTO(Java record가 잘 맞는다)로 매핑한다.

이유:

- 민감 필드 노출 (password hash 등)
- API와 DB 스키마가 결합됨
- 직렬화 중 `LazyInitializationException` 가능

## 4. JPA 연관 관계는 명시적으로 LAZY

`@ManyToOne`, `@OneToOne`의 기본값은 EAGER인데, EAGER은 한번 정하면 쿼리 시점에 LAZY로 못 바꾼다. **모든** 연관 관계에 `fetch = FetchType.LAZY`를 명시한다.

N+1 문제는 `JOIN FETCH` 또는 entity graph로 해결한다. `findAll()`을 컬렉션 매핑이 있는 엔티티에 그냥 호출하지 않는다.

## 5. JPA 엔티티에 Lombok @Data 쓰지 않는다

자동 생성된 `equals`/`hashCode`가 모든 필드 기준이라 Hibernate 프록시 동일성을 깬다. `@ToString`이 lazy 컬렉션을 건드려 LazyInitializationException을 일으킬 수 있다.

엔티티에 안전한 조합: `@Getter`, `@Setter`(필요한 곳만), `@NoArgsConstructor`(JPA가 요구), `@AllArgsConstructor`(필요 시). `equals`/`hashCode`는 PK 기반으로 손으로 쓰거나, `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + `@EqualsAndHashCode.Include`로 PK만 포함.

## 6. 예외 처리는 @RestControllerAdvice에 모은다

컨트롤러에서 `try/catch(Exception)` 흩어놓지 않는다. `@RestControllerAdvice` 한 곳에서 타입별 `@ExceptionHandler`로 처리하고 일관된 `ProblemDetail` (RFC 7807) 응답을 만든다.

## 7. 요청 DTO에 @Valid + Bean Validation

DTO 필드에 `@NotBlank`, `@Email`, `@Size` 등. 컨트롤러 파라미터에 `@Valid`. `MethodArgumentNotValidException`을 `@RestControllerAdvice`에서 422로 매핑.

검증 실패는 400이 아니라 **422**다.

## 8. 프로덕션에서 ddl-auto=update 금지

`spring.jpa.hibernate.ddl-auto`는 프로덕션에서 `validate` 또는 `none`. 스키마 변경은 Flyway나 Liquibase로 명시적 마이그레이션.

## 9. Package-by-feature

```
com.app.order.{ Order, OrderController, OrderService, OrderRepository, OrderDto }
com.app.user.{ ... }
com.app.payment.{ ... }
```

NOT `com.app.controller`, `com.app.service`, `com.app.repository`. feature 패키지 안에서는 package-private 가시성으로 캡슐화한다.

## 10. SOLID는 코드 리뷰의 기본 어휘

이름만 외우는 게 아니라 실제로 검토 기준으로 쓴다.

- **S**: 한 클래스가 두 가지 이유로 바뀐다면 분리한다.
- **O**: 새 동작은 새 클래스/인터페이스 구현으로. 기존 클래스의 `if/else` 추가가 아님.
- **L**: 서브클래스가 상위 타입의 약속을 깨면 상속 관계가 잘못된 것.
- **I**: 한 인터페이스에 메서드가 너무 많으면 작은 인터페이스로 쪼갠다.
- **D**: 도메인은 인터페이스(port)에 의존하고, 구현체는 인프라가 제공한다.

## 흔한 안티패턴

- 서비스가 다른 서비스 너무 많이 의존 → 유스케이스 분리 신호
- `Optional<T>`을 필드 타입으로 사용 (반환 타입에만 써야 함)
- `@Component` / `@Service` / `@Repository` 의미 없이 섞어 쓰기 — 의도에 맞게 쓴다
- `@ComponentScan` 범위를 너무 넓게 잡아 테스트가 느려짐
