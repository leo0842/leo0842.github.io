---
layout: single
title: "[JPA] 일대다(OneToMany) 테이블 생성과 데이터 삽입, 조회"
categories: Computer-Science
tags: [JPA, HIBERNATE, SPRING, SPRING_BOOT]

date: 2021-06-17
last_modified_at: 2021-06-17
---
이번 포스트는 [지난 포스트](https://leo0842.github.io/computer-science/%EB%8B%A4%EB%8C%80%EB%8B%A4(ManyToMany)-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EC%84%A4%EC%A0%95/) 에 이어서 작성되었습니다.

안녕하세요. 이번 포스트에서는 양방향 매핑의 연관 관계에 있는 테이블들에 데이터를 어떤 방식으로 삽입하고 조회하는지에 대해 알아보겠습니다.


## 의문점

1. 양방향 매핑이면 서로의 PK를 가지고 있어야 할텐데 새로 생성되는 데이터들에게는 서로의 PK를 어떻게 입력하지? 순차적으로 입력해야하나? 서로 동등한 위치라면 순서는 어떻게 해야하지?
2. 이렇게 순차적으로 서로의 PK를 저장한다고 해도 1:N이라면 1 테이블에서는 N 테이블의 PK가 리스트로 저장되는데 어떻게 표현되지?
3. Json 응답에서 무한 루프는 어떻게 해결해야할까?

## 세팅

```java
public class Post{
  
  //...

@OneToMany(mappedBy = "post")
private List<PostTag> tags;
  }
```
```java
public class Tag {
  
  //...
  
  @OneToMany(mappedBy = "tag")
  private List<PostTag> posts;
}
```
```java
public class PostTag {
  
  //...
  
  @ManyToOne
  private Post post;

  @ManyToOne
  private Tag tag;
}
```

현재 엔터티에 설정된 연관관계입니다. 엔터티에서 양방향으로 설정하였기때문에 DB에서도 테이블이 양방향으로 설정될 것으로 예상하였지만 생성된 테이블의 구성은 달랐습니다.

## 테이블 생성

현재 설정은 spring.jpa.hibernate.ddl-auto=create-drop 입니다.

    create table post (id bigint not null auto_increment, title varchar(255), primary key (id))
    
    create table post_tag (id bigint not null auto_increment, point integer, post_id bigint, tag_id bigint, primary key (id))
    
    create table tag (id bigint not null auto_increment, name varchar(255), primary key (id))


테이블 생성을 할 때는 연관 관계를 설정하지 않고 GenerationType, PK 등을 설정하는 것을 볼 수 있습니다. 또한 @ManyToOne 으로 조인을 걸었던 속성은 객체가 아닌 bigint 로 저장되었습니다.

    alter table post_tag add constraint FKc2auetuvsec0k566l0eyvr9cs foreign key (post_id) references post (id)
    
    alter table post_tag add constraint FKac1wdchd2pnur3fl225obmlg0 foreign key (tag_id) references tag (id)


이후 제약 사항 쿼리를 통해 각각 조인이 걸린 속성에 외래키를 설정합니다.

```bash
desc post;
```

| Field | Type         | Null | Key | Default | Extra          |
|-------|--------------|------|-----|---------|----------------|
| id    | bigint       | NO   | PRI | NULL    | auto_increment |
| title | varchar(255) | YES  |     | NULL    |                |

```bash
desc tag;
```

| Field | Type         | Null | Key | Default | Extra          |
|-------|--------------|------|-----|---------|----------------|
| id    | bigint       | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255) | YES  |     | NULL    |                |


그 결과 1:N 에서 1에 해당하는 post와 tag 테이블에는 매핑하였던 N 테이블의 정보가 없습니다. 그럼 N에 해당하는 테이블을 확인해보겠습니다.


```bash
desc post_tag;
```

| Field   | Type   | Null | Key | Default | Extra          |
|---|---|---|---|---|---|
| id      | bigint | NO   | PRI | NULL    | auto_increment |
| point   | int    | YES  |     | NULL    |                |
| post_id | bigint | YES  | MUL | NULL    |                |
| tag_id  | bigint | YES  | MUL | NULL    |                |

해당 테이블에는 ManyToOne 으로 설정한 속성이 foreign key 방식으로 각각의 pk가 담겨있습니다.

DB에서는 Join을 foreign key로 맺기때문에 참조하는 테이블에서만 해당 조인의 정보가 나타납니다. 
foreign key 만으로도 각각의 테이블에서 조인된 정보를 확인할 수 있기때문에 단방향으로 생성이 되고 연관관계의 주인(foreign key 가 저장된 테이블)이라는 정보가 만들어지게 됩니다.

ex) select * from A where A.id = B.A_id; 와 select * from B where A.id = B.A_id; 처럼 foreign key 하나로 각각 테이블에서 조인 정보 확인이 가능합니다.

위의 결과로부터 여러 정보를 알 수 있었습니다.
- ORM과 DB의 차이가 나타난다.
  
  ->  DB는 foreign key로 조인 정보를 저장하지만 Java 에서는 객체로 표현됩니다. 그리고 hibernate 와 같은 ORM 을 통해 서로 매핑이 됩니다.

![orm_db](/assets/images/orm_db.png)

- @ManyToOne 의 FetchType default 값이 EAGER 인 이유와 @OneToMany 의 FetchType default 값이 LAZY 인 이유를 알 수 있다.

  -> @ManyToOne 이 담긴 테이블은 조인된 정보를 이미 담고있기때문에 EAGER를 통해 가진 정보를 바로 가져옵니다.
  
  -> @OneToMany 가 담긴 테이블은 조인된 정보를 담고있지 않기때문에 조인된 정보가 필요없다면 굳이 한번에 가져 올 필요가 없어서 LAZY 가 default 입니다.

## 해결
의문점 1번과 2번이 한번에 해결되었습니다. JPA 에서 양방향으로 매핑을 했더라도 DB 에서는 단방향으로 표현이 됩니다. 따라서 서로의 PK를 가지고 있는 것도 아니고 1:N 에서 1 에 해당하는 테이블은 연관 관계의 리스트 정보를 가지고 있지도 않습니다.

## 데이터 삽입
테이블 생성의 플로우와 유사하게 1:N 에서 1에 해당하는 테이블의 객체를 먼저 만듭니다.

그리고 저장된 객체를 N에 해당하는 테이블의 객체의 속성에 set 한 뒤 저장합니다.
```java
@Service
@RequiredArgsConstructor
public class TestService {
  
  private final PostRepository postRepository;
  private final TagRepository tagRepository;
  private final PostTagRepository postTagRepository;

  Post post = postRepository.save(new Post());
  Tag tag = tagRepository.save(new Tag());
  PostTag postTag = postTagRepository.save(PostTag.builder().post(post).tag(tag).build());
  
  //...
}

```
객체를 생성할 때 테스트를 위해 @Builder 를 사용하였습니다.

발생한 쿼리를 살펴보겠습니다.

    insert into post (title) values (?)
    insert into tag (name) values (?)
    insert into post_tag (point, post_id, tag_id) values (?, ?, ?)

에상했던대로 post 와 tag 테이블에 데이터를 삽입한 뒤 이 데이터들의 id 를 저장하는 쿼리가 발생하였습니다.

** 여기서 레포지토리에 저장이 되지 않은 post 또는 tag 객체를 post_tag 의 객체에 넣고 저장한다면 foreign key 에 해당하는 post 또는 tag 의 id 가 담기지 않습니다.
이 데이터의 pk 는 auto_increment 이기때문에 저장되지 않았을 때에는 id 가 생성되지 않았기 때문입니다.

DB 에 저장된 데이터를 확인하겠습니다.

| id | name |
|----|------|
|  1 | name |


| id | title |
|----|------|
|  2 | title |

| id | point | post_id | tag_id |
|----|------|------|------|
|  3 |  NULL |       2 |      1 |

이상없이 데이터가 저장되었습니다. 이제 저장된 데이터를 조회하는 방법과 주의점을 알아보겠습니다.

## 데이터 조회 

조인된 데이터를 조회할 때는 FetchType 으로 LAZY 와 EAGER 를 API 특성에 맞게 설정을 하여 효율적인 쿼리를 목표로 합니다. LAZY 와 EAGER 에 대한 더욱 자세한 내용은 인터넷에 풍부하기때문에 이번 포스트에서는 간단하게 쿼리만 살펴보겠습니다.

조회하여 응답을 내보낼 때에는 매핑으로 인해 무한 루프가 발생합니다. 
따라서 dto 또는 @JsonIgnore 어노테이션을 이용하여 응답을 내보냅니다. 
다만 API 의 통일된 형태와 다양한 응답으로 인해 해당 속성을 참조해야 하는 응답이 있을 수 있습니다. 
그래서 @JsonIgnore 보다는 dto 를 선호한다고 합니다.

```java
@Data
public class PostResponseDto {

  String title;

  List<String> tags;

  public PostResponseDto(Post post) {
    this.title = post.getTitle();
    this.tags = post.getTags().stream().map(postTag -> postTag.getTag().getName()).collect(Collectors.toList());
  }
}
```
stream 을 이용하여 tag name 리스트를 반환하게 만들었습니다.

```java
@RestController
@RequiredArgsConstructor
public class TestController {

  private final PostRepo postRepo;

  @GetMapping("/test")
  public PostResponseDto test() {
    Post post = postRepo.findById(1L).orElse(null);
    if (post != null) {
      return new PostResponseDto(post);
    }
    return null;
  }
}
```
    select 
          post0_.id as id1_1_0_, 
          post0_.title as title2_1_0_

    from 
          post post0_

    where     
          post0_.id=?
lazy 라서 tag 쿼리 발생이 되지 않았습니다. 

    select 
           tags0_.post_id as post_id3_2_0_, 
           tags0_.id as id1_2_0_, 
           tags0_.id as id1_2_1_, 
           tags0_.point as point2_2_1_, 
           tags0_.post_id as post_id3_2_1_, 
           tags0_.tag_id as tag_id4_2_1_, 
           tag1_.id as id1_3_2_, 
           tag1_.name as name2_3_2_ 

    from 
           post_tag tags0_ 

    left outer join 
           tag tag1_ on tags0_.tag_id=tag1_.id 

    where 
           tags0_.post_id=?
getTags 메소드가 호출될 때 조인 쿼리가 발생하고 해당 엔터티의 Tag 는 EAGER 이기때문에 곧바로 조인 쿼리가 발생하였습니다.

    {"title":"title","tags":["name"]}

