---
layout: single
title: "[Spring] ArgumentResolver 커스텀하기"
categories: [Spring]
tags: [SPRING, SPRING_BOOT]

date: 2021-08-04
last_modified_at: 2021-08-04
---

안녕하세요. 오늘은 HandlerMethodArgumentResolver 인터페이스를 커스텀하는 방법을 소개하겠습니다.

## 계기

진행하던 프로젝트에서는 요청한 유저의 식별을 헤더의 jwt 토큰을 통해 확인하였습니다.

처음에는 filter 단에서 jwt 를 복호화하여 Authentication 형태로 내려주고,

컨트롤러에서 Authentication 을 받은 뒤 claims 를 확인하여 유저를 식별하였습니다.

이러한 방식에는 두 가지 문제가 있었습니다.

1. 컨트롤러가 많아질수록 반복되는 로직이 많아진다.

매 컨트롤러마다 Authentication 을 파라미터로 받고, claims 를 받는 로직이 반복되었습니다.

그래서 파라미터와 claims 를 받는 로직을 앞에서 공통으로 해주고, 필요로하는 user 의 id 만 받고자 하였습니다.

2. 에러 핸들링에 문제가 생겼다.

컨트롤 할 에러가 많아져 Advice 를 이용하여 에러를 한 곳에서 처리하고 응답을 같은 형식으로 내려보내려고 하였습니다.

하지만 filter 단에서 jwt 토큰을 처리하면 발생하는 에러에 대해 advice 에서 처리하지 못해서 filter 뒤에서 처리하고자 하였습니다.

## 구현

그래서 찾아보던 중 발견한 것이 Argument Resolver 입니다.

### Argument Resolver

Argument Resolver 는 filter 와 interceptor 이후, controller 이전에 동작됩니다.

controller 에서 받는 파라미터에 대해 공통으로 처리가 필요할 때 Argument Resolver 를 사용합니다.

먼저 기본적으로 처리되고 있는 Argument Resolver 에 대해서 알아보겠습니다.

```java

public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {

  // ...
  
  private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

    // Annotation-based argument resolution
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    resolvers.add(new MatrixVariableMethodArgumentResolver());
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    resolvers.add(new ServletModelAttributeMethodProcessor(false));
    resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    resolvers.add(new RequestAttributeMethodArgumentResolver());

    // Type-based argument resolution
    resolvers.add(new ServletRequestMethodArgumentResolver());
    resolvers.add(new ServletResponseMethodArgumentResolver());
    resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    resolvers.add(new ModelMethodProcessor());
    resolvers.add(new MapMethodProcessor());
    resolvers.add(new ErrorsMethodArgumentResolver());
    resolvers.add(new SessionStatusMethodArgumentResolver());
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
    if (KotlinDetector.isKotlinPresent()) {
      resolvers.add(new ContinuationHandlerMethodArgumentResolver());
    }

    // Custom arguments
    if (getCustomArgumentResolvers() != null) {
      resolvers.addAll(getCustomArgumentResolvers());
    }

    // Catch-all
    resolvers.add(new PrincipalMethodArgumentResolver());
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    resolvers.add(new ServletModelAttributeMethodProcessor(true));

    return resolvers;
  }
  
  // ...

}
```

디폴트 값으로 꽤 많은 Argument Resolver 가 등록되어 있는 것을 확인할 수 있습니다.

Annotation-based 에 우리가 자주 사용하는 pathVariable 에 대한 Resolver, Request 와 Response Body 에 대한 Resolver 도 눈에 띕니다.

Argument Resolver 에 대해 조금 더 자세한 내용은 [Spring Argument Resolver](https://blog.neonkid.xyz/238) 글을 참조해주세요!

이제 우리만의 커스텀 Argument Resolver 를 만드는 방법에 대해서 알아보겠습니다.

```java

@Retention(RetentionPolicy.RUNTIME) // --- 1
@Target(ElementType.PARAMETER) // --- 2
public @interface UserInfo {

}
```

저는 auth 라는 패키지 안에 해당 어노테이션 클래스와 아래의 Custom Resolver 를 넣어주었습니다.

- 1번 Retention 어노테이션은 해당 어노테이션 클래스의 메모리를 어디까지 가져갈 지에 대한 설정입니다. [@Retention 어노테이션 까보기](https://sas-study.tistory.com/329?category=817408) 글에서 Retention 어노테이션에 대해 더 자세한 설명을 볼 수 있습니다.

- 2번 Target 어노테이션은 어노테이션 타입이 어떤 문맥(Contexts)에서 적용될 지를 정하는 어노테이션입니다. enum 클래스로 METHOD, FIELD 등이 있고 본 포스트에서는 파라미터로 활용할 계획이기에 PARAMETER 로 설정합니다.

```java

@Component
public class UserInfoArgumentResolver implements HandlerMethodArgumentResolver {

  @Override
  public boolean supportsParameter(MethodParameter parameter) { // --- 1
    return parameter.getParameterAnnotation(UserInfo.class) != null && // --- 1.1
        String.class.equals(parameter.getParameterType()); // --- 1.2
  }

  @Override
  public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
      WebDataBinderFactory binderFactory) throws Exception { // --- 2

    return webRequest.getHeader("Host");
  }
}
```

같은 auth 패키지 내부에 Custom 할 Argument Resolver 클래스를 생성하고 HandlerMethodArgumentResolver 인터페이스를 implements 를 해줍니다.

그리고 컴포넌트 어노테이션을 통해 빈으로 등록하였습니다.

- 1번 supportsParameter 메소드는 커스텀하여 만든 어노테이션 파라미터가 있는지 확인하는 메소드입니다.
  
  - 1.1번은 이름이 UserInfo 라는 어노테이션이 있는지 확인합니다.
  - 1.2번은 받는 타입이 여기서 지정한 타입과 같은지 확인합니다. 이번 예시에서는 만들 파라미터 어노테이션이 String 이기때문에 String.class.equals() 를 하였습니다.
  

- 2번 resolveArgument 메소드는 해당 파라미터 어노테이션에서 어떤 값이 반환될 지에 대한 메소드입니다. 
  이번 예시에서는 헤더의 "Host" 를 가져와 반환합니다. 
  실제 프로젝트에서는 헤더의 Authorization 을 가져와 정제하고 복호화한 뒤 유저의 ID 를 반환하게 하였습니다.

이제 작성한 커스텀 Argument Resolver 를 설정 파일에 추가하겠습니다.

```java

@Configuration
@RequiredArgsConstructor
public class ConfigImpl implements WebMvcConfigurer {

  private final UserInfoArgumentResolver userInfoArgumentResolver;

  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(userInfoArgumentResolver);
  }
}
```

저는 따로 설정 클래스를 만들고 Configuration 빈으로 등록하였습니다.

WebMvcConfigurer 를 상속받으면 addArgumentResolvers 를 오버라이딩할 수 있고, 직접 위에서 작성한 커스텀 ArgumentResolver 를 add 합니다.

## 테스트

```java

@WebMvcTest(TestController.class)
class TestControllerTest {

  @Autowired
  private MockMvc mvc;

  @Test
  @DisplayName("Argument Resolver 구동 확인")
  public void test() throws Exception {

    mvc.perform(get("/test")
        .header("Host", "Test!!"))
        .andExpect(status().isOk())
        .andExpect(content().string("Test!!"));
  }

}
```

헤더에 테스트용 문자를 넣고 테스트를 실행해 보았습니다.

![테스트 성공](/assets/images/ArgumentResolver_test_passed.png)

실제로 로컬 서버를 열어서 확인도 해 보았습니다!

![로컬 테스트 성공](/assets/images/local_test_passed.png)

로컬 서버의 호스트인 127.0.0.1:8080 이 반환되는 것을 확인할 수 있습니다.