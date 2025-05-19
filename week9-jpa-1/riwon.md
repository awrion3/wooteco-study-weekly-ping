### @GeneratedValue

```java
@Entity
public class Theme {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    //...
}
```

@GeneratedValue는 엔티티의 기본 키 값을 자동으로 생성해주는 어노테이션이며, 주로 @Id와 함께 사용된다.
그리고 이때 strategy 속성은 어떤 방식으로 ID를 생성할지를 아래와 같이 정의한다.

|전략|설명|
|:-:|:-:|
|IDENTITY|데이터베이스의 auto-increment 기능을 사용|
|SEQUENCE|JPA가 시퀀스 객체를 이용해 ID를 생성|
|TABLE|키 생성 전용 테이블을 만들어 ID 관리|
|AUTO|데이터베이스에 맞게 자동 선택|

1. GenerationType.IDENTITY
    - 데이터베이스에 기본 키 생성을 위임.
    - 주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용. (예: MySQL AUTO_INCREMENT)
    - em.persist()로 객체를 영속화 시키는 시점에 insert 쿼리가 DB로 전송되고, 
      반환받은 id 값을 가지고 영속성 컨텍스트에 엔티티를 등록시켜 관리한다.


2. GenerationType.SEQUENCE
    - 데이터베이스 시퀀스를 활용하여 id 값을 증가.
    - 주로 오라클, PostgreSQL, DB2, H2 데이터베이스에서 사용.
    - 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트이다. (예: 오라클 시퀀스)


3. GenerationType.TABLE
    - 키 생성 전용 테이블을 만들어서 데이터베이스 시퀀스를 흉내내는 전략.
    - 모든 데이터베이스에 적용 가능하나, 성능적인 손해가 있어서 잘 쓰지 않는다.


4. GenerationType.AUTO
    - `hibernate.dialect`에 설정된 DB 방언 종류에 따라, 하이버네이트가 전략을 선택하게끔 위임.

---

### FetchType

아래와 같이 Team과 Member는 양방향 연관관계가 존재하고, 연관관계의 주인은 Member라고 가정한다.

```java
@Entity
public class Member {
    
    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @Column(name = "username")
    private String username;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "team_id")
    private Team team;
}
```

```java
@Entity
public class Team {

    @Id
    @GeneratedValue
    @Column(name = "team_id")
    private Long id;

    @Column(name = "name")
    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```

그리고 Team 객체와 Member 객체를 각각 만들고, Member 객체의 경우 Setter 메소드를 통해 Team 객체를 셋팅해준다.
이후 em.find() 메소드를 통해 Member를 조회한다고 가정한다.

```java
//...
Team team = new Team();
team.setName("teamA");
em.persist(team);

Member member = new Member();
member.setUsername("member1");
member.setTeam(team);
em.persist(member);

em.flush();
em.clear();

Member findMember = em.find(Member.class, member.getId());
```

이때 FetchType.EAGER인 경우 쿼리는 아래와 같다.

```
Hibernate: 
    select
        member0_.MEMBER_ID as MEMBER_I1_0_0_,
        member0_.TEAM_ID as TEAM_ID3_0_0_,
        member0_.USERNAME as USERNAME2_0_0_,
        team1_.TEAM_ID as TEAM_ID1_1_1_,
        team1_.name as name2_1_1_ 
    from
        Member member0_ 
    left outer join
        Team team1_ 
            on member0_.TEAM_ID=team1_.TEAM_ID 
    where
        member0_.MEMBER_ID=?
```

또한 FetchType.LAZY인 경우 쿼리는 아래와 같다.

```
Hibernate: 
    select
        member0_.MEMBER_ID as MEMBER_I1_0_0_,
        member0_.TEAM_ID as TEAM_ID3_0_0_,
        member0_.USERNAME as USERNAME2_0_0_ 
    from
        Member member0_ 
    where
        member0_.MEMBER_ID=?
```

이처럼 EAGER는 Member를 조회하면 연관관계에 있는 Team 역시 함께 조회하는 반면에, 
LAZY는 Member만 조회해오고 연관관계에 있는 나머지 데이터는 조회를 미룬다는 특성이 있다.

---
