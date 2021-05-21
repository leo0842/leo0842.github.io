---
layout: single
title: "[JPA] 다대다(ManyToMany) 연관관계를 설정할 때는?"
categories: Computer-Science
tags: [JPA, SPRING, SPRING_BOOT]

date: 2021-03-21
last_modified_at: 2021-03-21
---

안녕하세요. 이번 포스트에서는 JPA를 이용한 다대다 연관관계(ManyToMany)에 대하여 이야기를 해보려고 합니다.

개발을 하다보면 다대다 연관관계를 설정해야 하는 경우가 언제나 생깁니다.
처음에 연습하면서 공부를 할 때에는 이정도 쯤이야 껌이지 라는 생각으로 단순히 @ManyToMany 어노테이션을 붙이고 사용하였습니다.
하지만 프로젝트를 하면서 다대다 연관관계 설정을 위해 조금 더 자세히 찾아보면서 단순히 @ManyToMany 어노테이션만 사용하면 한계점이 분명히 생긴다는 것을 알게 되었습니다. 

지금부터 한계점과 대안을 알아보도록 하겠습니다. 먼저 예시 엔터티를 만들겠습니다.

## 다대다(@ManyToMany)

```java
@Entity
public class Post {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;
  
  private String body;

  @ManyToMany
  @JoinTable(
    name = "tags_to_posts",
    joinColumns = @JoinColumn(name = "post_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id"))
  private List<Tag> tags;

}
```

```java
@Entity
public class Tag {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  @ManyToMany(mappedBy = "tags")
  private List<Post> posts;

}
```

게시글과 태그에 대한 테이블을 양방향으로 간단하게 만들어 보았습니다. 

다대다 관계를 맺을 때에는 데이터베이스의 특성에 의해서 새로운 테이블이 필요합니다.
그래서 JoinTable과 JoinColumn 어노테이션을 붙여주는데, JPA에서는 해당 어노테이션이 없어도 자동으로 만들어주기때문에 없애도 무방한 어노테이션입니다.
다만 JPA가 자동으로 생성해주는 테이블 명과 컬럼 명이 컨벤션과 언제나 일치하는 것은 아니기 때문에 가능하면 해당 어노테이션을 명시하는 것이 좋습니다.
그래서 꼭 필요한 것은 연결을 시킬 컬럼 명을 mappedBy 파라미터로 적어주는 것입니다.

## 한계점과 대안


위와 같이 JPA에서 직접 만들어주는 테이블이 단순히 두 테이블을 연결시키기 위한 테이블이라면 매우 편리할 수 있습니다. 더이상 관리를 할 필요가 없기 때문이죠.
하지만 연결 테이블이 관계를 위한 id 컬럼 말고도 다른 정보를 담아야 한다면 어떨까요?
ManyToMany에 의해 만들어진 연결 테이블은 매핑 정보만 저장을 하고 직접 관리할 수 없기때문에 다른 컬럼을 생성할 수 없는 한계가 있습니다.


예를 들어 포스트에 붙은 여러 태그들에 포스트와의 연관성을 점수로 매겨 저장한다고 생각하면 연결 테이블에서 해당 정보를 담는 것이 가장 직관적이고 효율적입니다.
그래서 이에 대한 대안으로 ManyToMany를 두 개의 OneToMany로 만들고 자동으로 만들어졌던 테이블을 직접 관리할 수 있는 엔터티로 만들어 관리할 수 있습니다. 
구도는 매우 유사하지만 엔터티가 하나 늘은 대신 우리가 직접 관리를 하고 컬럼을 생성하거나 필요하면 메소드도 생성할 수 있게 됩니다.

```java
@Entity
public class PostTag {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private Integer relationPoint;
  
  private Integer importancePoint;

  @ManyToOne
  private Post post;

  @ManyToOne
  private Tag tag;
  
  public Integer calPoint() {
    return relationPoint * importancePoint;
  }
  
}
```
먼저 두 테이블의 중간에서 연관 관계를 관리해줄 연결 테이블을 만들어줍니다. 
이 테이블 역시 하나의 엔터티이므로 id를 가지게 되고 조인될 두 테이블을 저장할 속성에 ManyToOne 어노테이션을 달아주어 명시해줍니다.
또한 추가적으로 관리할 속성들 또는 메소드들을 정의하고 사용할 수 있습니다.
```java
public class Post{
  
  ...

@OneToMany(mappedBy = "post")
private List<PostTag> tags;
  }
```
```java
public class Tag {
  
...
  
  @OneToMany(mappedBy = "tag")
  private List<PostTag> posts;
}
```
위의 두 테이블에서 연관 관계를 맺을 속성을 OneToMany로 바꿔 조인을 걸어줍니다.

이와같이 다대다 연관 관계를 설정을 할 때에는 두 개의 OneToMany로 관계를 걸어주고 연결 테이블을 생성하여 더욱 효율적으로 관리할 수 있게 됩니다.

다음 포스트에서는 해당 조인 관계를 직접 사용하여 인스턴스 생성과 조회, 주의점을 알아보도록 하겠습니다.
