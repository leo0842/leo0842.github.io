---
layout: single
title: "[Spring] EnableJpaAuditing 맛보기"
categories: [Spring]
tags: [SPRING, SPRING_BOOT]

date: 2021-07-21
last_modified_at: 2021-07-21
---

안녕하세요. 오늘은 EnableJpaAuditing 어노테이션에 대하여 알아보겠습니다.

먼저 EnableJpaAuditing 를 가장 자주 만나는 [생성일자, 수정일자 자동으로 등록](#생성일자와-수정일자-자동-등록)하는 방법을 보고 [EnableJpaAuditing 가 가지는 여러 인자](#EnableJpaAuditing)에 대해 알아보겠습니다.

## 생성일자와 수정일자 자동 등록

객체를 생성하거나 수정할 때 생성자와 Setter 에 LocalDateTime.now() 등 시간을 나타내는 객체를 넣어 생성일자와 수정일자를 관리할 수 있습니다.

하지만 여러 엔터티에서 이러한 코딩을 매번 하는 것은 단순하고 번거로운 작업입니다.

그래서 인스턴스를 생성하거나 수정하는 변화가 있을 시에 이를 감지하여 자동으로 일시를 저장하도록 할 수 있습니다.

### 엔터티

```java

@Entity
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Builder
@EntityListeners(AuditingEntityListener.class) // --- 1
public class Board {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;

  @Setter
  private String content;

  @CreatedDate // --- 2
  private LocalDateTime createdAt;

  @LastModifiedDate // --- 3
  private LocalDateTime updatedAt;

}
```

1. EntityListeners

    * 엔티티를 DB에 적용하기 이전, 이후에 콜백 리스너를 요청할 수 있는 어노테이션
    * EntityListeners 에 대해 더욱 많은 자료를 보고싶으신 분은 [JPA entity listener 만들기](https://bum752.github.io/posts/JPA-entity-listener-%EB%A7%8C%EB%93%A4%EA%B8%B0/) 를 참고해주세요!
  
2. CreatedDate
    * 인스턴스가 생성되는 것을 감지하여 감지된 일자를 저장합니다.
  
3. LastModifiedDate
    * 인스턴스가 수정되는 것을 감지하여 감지된 일자를 저장합니다.
  
### 레포지토리

```java

public interface BoardRepository extends JpaRepository<Board, Long> {

}
```

레포지토리는 JpaRepository 를 상속받겠습니다.

### MainApplication

```java

@SpringBootApplication
@EnableJpaAuditing // --- 1
public class BlogApplication {

  public static void main(String[] args) {

    SpringApplication.run(BlogApplication.class, args);
  }
}
```

1. EnableJpaAuditing

    * Configuration 어노테이션을 통해 JPA 에서 auditing 을 가능하게 하는 어노테이션

### 테스트 기본 세팅

```java

@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class BoardRepositoryTest {

  @Autowired
  private BoardRepository boardRepository;

  String title;

  String content;

  Board board;

  @BeforeEach
  public void setUp() {

    title = "hi";
    content = "hello";
    board = boardRepository.save(Board.builder().title(title).content(content).build());
  }
}
```

* 같은 동작이 많아 BeforeEach 를 통해 save() 메소드가 매 테스트마다 동작하게 하였습니다.

### 일자 자동 저장 테스트

```java
class BoardRepositoryTest {
  
  // ...

  @Test
  @DisplayName("생성일자 수정일자 자동 등록 확인")
  public void auditTest() {

    assertThat(board.getUpdatedAt(), is(not(nullValue())));

    System.out.println(board.getCreatedAt());
    System.out.println(board.getUpdatedAt());
  }
}
```

이미 setUp 에서 board 를 저장하였으므로 바로 assertThat 으로 확인하고 날짜를 찍어보았습니다.

![audit 테스트 성공](/assets/images/audit_test_passed.png)

builder 에 createdAt 과 updatedAt 에 변수를 넣지 않았음에도 nullValue 가 아닌 것을 확인하였습니다.

createdAt 과 updatedAt 에 데이터가 잘 들어간 것도 확인 할 수 있습니다.

## EnableJpaAuditing

생성일자와 수정일자를 자동으로 등록할 수 있는 기능 외에도 EnableJpaAuditing 에는 다양한 기능이 있습니다.

```java

@Inherited
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(JpaAuditingRegistrar.class)
public @interface EnableJpaAuditing {

	String auditorAwareRef() default "";

	boolean setDates() default true;

	boolean modifyOnCreate() default true;
	
	String dateTimeProviderRef() default "";
}
```

이 중 auditorAwareRef 는 @CreatedBy, @LastModifiedBy 과 함께 생성한 사람, 수정한 사람을 자동으로 저장해줍니다.

이 인자는 User 에 대한 정보가 있어야 하기 때문에 auditorAwareRef 인자에 대해 더욱 알아보고 싶은 분들은 [Auditing with JPA, Hibernate, and Spring Data JPA](https://www.baeldung.com/database-auditing-jpa) 를 확인해주세요!

### setDates

setDates 인자는 생성일자와 수정일자를 설정할 것인지에 대한 boolean 값입니다.

default 값은 true 이고 false 값을 넣어 확인해보겠습니다.

```
@EnableJpaAuditing(setDates = false)
```

```java

@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class BoardRepositoryTest {
  
  // ...

  @Test
  @DisplayName("setDates 인자 확인")
  public void setDates() {

    assertThat(board.getUpdatedAt(), is(nullValue()));

    System.out.println(board.getCreatedAt());
    System.out.println(board.getUpdatedAt());
  }
}
```

![setDates 테스트 성공](/assets/images/setDates_test_passed.png)

false 로 설정하면 null 인 것을 확인할 수 있습니다.

### modifyOnCreate

modifyOnCreate 인자는 생성시 수정일자를 함께 등록하는지 설정하는 인자입니다.

default 값은 true 이고 false 값을 넣어 확인해보겠습니다.

```
@EnableJpaAuditing(modifyOnCreate = false)
```

```java

@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class BoardRepositoryTest {

  @Autowired
  private EntityManager em;

  // ...

  @Test
  @DisplayName("modifyOnCreate 인자 확인")
  public void modifyOnCreate() throws InterruptedException {

    assertThat(board.getUpdatedAt(), is(nullValue()));

    System.out.println(board.getCreatedAt());
    System.out.println(board.getUpdatedAt());

    board.setContent("bye");

    Thread.sleep(3000);

    em.flush();
    em.clear();

    assertThat(board.getUpdatedAt(), is(not(nullValue())));

    System.out.println(board.getCreatedAt());
    System.out.println(board.getUpdatedAt());
  }
}
```

저장 적용시키기위해 EntityManager 를 받아 flush() 와 clear 를 실행하였고 사이 시간을 주기 위해 3초를 재웠습니다.

전과 후를 비교하여 createdAt 은 그대로이고 updatedAt 만 null 에서 수정된 시간이 저장되어야 합니다.

![modifyOnCreate 테스트 성공](/assets/images/modifyOnCreate_test_passed.png)

성공한 것을 확인할 수 있습니다.

그리고 수정 전과 후를 비교하였을 때 수정 전 생성할 때는 updatedAt 값이 null 인 것도 확인할 수 있습니다.

### dateTimeProviderRef

dateTimeProviderRef 인자는 저장하는 시간에 대해 커스터마이징할 수 있는 설정입니다.

dateTimeProviderRef 인자는 auditorAwareRef 와 마찬가지로 특정 인터페이스를 상속받아 메소드를 override 한 뒤 해당 클래스를 빈으로 등록해야 합니다.

EnableJpaAuditing 어노테이션의 설명에 DateTimeProvider 인터페이스를 상속하여 커스터마이징한다고 나와있습니다.

```java

@Component
public class DateTimeImpl implements DateTimeProvider {

  public DateTimeImpl() {
  }

  @Override
  public Optional<TemporalAccessor> getNow() {
    
    return Optional.of(LocalDateTime.now().minusHours(3L));
  }
}
```

DateTimeProvider 를 상속받아 getNow() 라는 메소드를 override 하고 @Component 를 통해 빈으로 등록합니다. 

3시간 시차가 나는 지역의 시간을 저장하도록 해보겠습니다.

```
@EnableJpaAuditing(dateTimeProviderRef = "dateTimeImpl")
```

dateTimeProviderRef 인자에 해당 클래스의 빈 이름으로 설정합니다.

#### NoSuchBeanDefinitionException 에러

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class BoardRepositoryTest {

  @Test
  @DisplayName("dateTimeProviderRef")
  public void dateTimeProviderRef() {

    assertThat(board.getUpdatedAt(), is(not(nullValue())));

    System.out.println(board.getCreatedAt());
    System.out.println(board.getUpdatedAt());
  }
}
```

> org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'dateTimeImpl' available

이 때 주의할 것이 있습니다.

이전과 마찬가지의 세팅으로 테스트를 돌리면 빈이 등록이 되지 않았다는 에러가 발생합니다.

@Import 를 해봐도, 다시 한번 빈으로 등록했는지 확인해봐도 같은 에러가 발생하였고 다른 방법으로 해결할 수 있었습니다.

테스트 환경에서 바꿔야 하는 걸 알았고 DataJpaTest 에 설정을 추가해 주었습니다.

```
@DataJpaTest(includeFilters = @Filter(
    type = FilterType.ASSIGNABLE_TYPE,
    classes = {DateTimeImpl.class}
))
```

[참고한 해결 방법](https://github.com/spring-projects/spring-boot/issues/13337)

![dateTimeProviderRef 테스트 성공](/assets/images/dateTimeProviderRef_test_passed.png)

현재 시간에서 3시간 전의 시간이 저장된 것을 확인할 수 있습니다.
