---
layout: single
title: "[JPA] JpaRepository Test 수행 방법"
categories: [JPA]
tags: [JPA, JPA_REPOSITORY, TEST, SPRING, SPRING_BOOT]

date: 2021-07-20
last_modified_at: 2021-07-20
---

안녕하세요. 이번에는 JpaRepository 를 상속받은 Repository 의 수행 방법과 여러 에러들에 대해 알아보겠습니다.

> 개발 환경
> -  IntelliJ
> -  Spring Boot 2.4.5
> -  Gradle
> -  Lombok
> -  MySQL, h2

하나의 프로젝트를 진행하면서 테스트 수행을 제일 게을리 한 부분이 레포지토리에 대한 테스트였습니다.
기본적으로 테스트를 진행하기도 전에 에러가 많이 나기도 했고 MySQL 과의 테스트 연동 등에 대해서도 어려움이 있어 피하기만 하다가 실행이라도 되도록 기본은 갖추자는 생각으로 여러 시도를 하게 되었습니다.

---

## 기본 세팅
```java

@Entity
@AllArgsConstructor
@NoArgsConstructor
@Getter
public class Board {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;

  private String content;
}
```
```java

class BoardRepositoryTest {

  @Autowired
  private BoardRepository boardRepository;

  @Test
  public void boardTest() {

    String title = "hi";
    String content = "hello";

    Board board = new Board(title, content);

    boardRepository.save(board);

    assertThat(board.getId(), is(1L));
    assertThat(board.getTitle(), is(title));
  }
}
```

아무 어노테이션을 입히지 않고 테스트를 돌려 보았습니다.
당연하게도 NullPointerException 이 발생합니다.
돌려보기도 전에 @Autowired 에서 "Autowired members must be defined in valid Spring bean" 이라고 노란 줄로 친절하게 알려줍니다.

이제 @DataJpaTest 어노테이션을 붙이고 실행해보겠습니다.

```java

@DataJpaTest
class BoardRepositoryTest {

  // ...
}
```

빈으로는 등록이 되었고 NullPointerException 은 넘어갔지만 다른 에러가 발생합니다.
Repository Test 를 수행할 때 두 가지 경우에 대해서 알아보겠습니다.

---

## h2로 테스트를 수행하고자 하는 경우

실제 사용하는 DB 와는 상관없이 레포지토리의 메소드에 대해 스프링에 내장된 h2로 테스트를 간단하게 수행하고자 할 때가 있습니다.

- __h2 의존성을 추가하지 않은 경우__

h2 의존성을 추가하지 않은 채로 위의 테스트를 수행하면 아래와 같은 에러가 발생합니다.

> java.lang.IllegalStateException: "Failed to replace DataSource with an embedded database for tests. If you want an embedded database please put a supported one on the classpath or tune the replace attribute of @AutoConfigureTestDatabase."

바라보는 DataSource 설정이 MySQL 등 디폴트로 설정된 DB와 달라 발생합니다. 이에 대해서는 아래 MySQL 파트에서 더 알아보도록 하겠습니다.

> implementation 'com.h2database:h2'

h2 의존성을 추가하면 위의 에러를 피할 수 있지만 다른 에러가 발생합니다.

- __테스트에 대한 설정 파일이 부재한 경우__

테스트 시 테스트 모듈 내부에 설정 파일이 없으면 main 의 설정 파일을 읽습니다. 
따라서 @DataJpaTest 에서 디폴트 값으로 등록한 h2 드라이버와 main db 의 드라이버가 다를 시 sql 문법 에러를 반환합니다.

> org.hibernate.exception.SQLGrammarException: could not prepare statement

따라서 이 에러는 간단하게 test 모듈 내부에 resource 디렉토리를 생성한 뒤 application.yml yaml 파일을 생성하고 드라이버에 대한 정보를 적어주면 됩니다.

```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
```

![h2 테스트 성공](/assets/images/h2_test_passed.png)

테스트가 성공한 것을 확인할 수 있습니다.

---

## MySQL로 테스트를 수행하고자 하는 경우

해당 파트는 [[JPA] DataJpaTest: 테스트시에 실제 DB 사용하기](https://kangwoojin.github.io/programing/auto-configure-test-database/) 을 보고 참고하였고 링크의 글에서 더욱 자세하고 깔끔하게 나와있습니다!

h2 추가 없이 계속 사용하던 MySQL DB로 테스트를 수행하고자 할 때도 있습니다.
하지만 앞서 서술했듯이 h2 의존성이 없으면 DataSource 를 찾을 수 없다는 에러가 발생합니다.
test 의 resource 디렉토리에 MySQL에 대한 설정을 작성해도 마찬가지로 뜹니다.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
@Transactional
@AutoConfigureCache
@AutoConfigureDataJpa
@AutoConfigureTestDatabase
@AutoConfigureTestEntityManager
@ImportAutoConfiguration
public @interface DataJpaTest {

// ...
}
```

DataJpaTest 어노테이션에는 설정을 자동으로 해주는 많은 어노테이션이 달려있습니다. 
여기서 @AutoConfigureTestDatabase 를 보겠습니다.

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {

  @PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
  Replace replace() default Replace.ANY;

  EmbeddedDatabaseConnection connection() default EmbeddedDatabaseConnection.NONE;

  // ...
}
```

- replace 를 설정할 수 있고 디폴트 값은 ANY 입니다.
- connection 을 설정할 수 있고 디폴트 값은 NONE 입니다.

두 값 모두 enum 클래스이고 이 부분을 설정하는 것에 따라 값이 달라질 수 있습니다.
EmbeddedDatabaseConnection enum 클래스를 보겠습니다.

```java

public enum EmbeddedDatabaseConnection {

  NONE(null, null, null, (url) -> false),

  H2(EmbeddedDatabaseType.H2, DatabaseDriver.H2.getDriverClassName(),
    "jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE", (url) -> url.contains(":h2:mem")),

  DERBY(EmbeddedDatabaseType.DERBY, DatabaseDriver.DERBY.getDriverClassName(), "jdbc:derby:memory:%s;create=true",
    (url) -> true),

  @Deprecated
  HSQL(EmbeddedDatabaseType.HSQL, DatabaseDriver.HSQLDB.getDriverClassName(), "org.hsqldb.jdbcDriver",
    "jdbc:hsqldb:mem:%s", (url) -> url.contains(":hsqldb:mem:")),

  HSQLDB(EmbeddedDatabaseType.HSQL, DatabaseDriver.HSQLDB.getDriverClassName(), "org.hsqldb.jdbcDriver",
    "jdbc:hsqldb:mem:%s", (url) -> url.contains(":hsqldb:mem:"));

  // ...
}
```

이넘 클래스에는 H2, DERBY, HSQLDB 등이 있고 MySQL DB 에 대한 정보가 없습니다. 
따라서 h2 를 의존성에 추가하고 설정 파일에 추가하면 해당 h2 db 로 테스트를 진행합니다.
MySQL 로 설정 파일을 작성했다면 찾을 수 없기 때문에 에러가 발생합니다.
앞서 @AutoConfigureTestDatabase 에서 Replace 값이 ANY 이기때문에 Embedded Database 를 찾게 된 것입니다.  
그래서 해당 Embedded Database 를 쓰지 않도록 Replace 값에 NONE 으로 설정하면 우리가 사용하는 실제 Database 를 사용할 수 있습니다.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class BoardRepositoryTest {

  // ...
}
```

![mysql 테스트 성공](/assets/images/mysql_test_passed.png)
