- 부록1 정리

  ## 1. ToOne에는 fetch join이 잘 맞는다

  `ManyToOne`, `OneToOne` 같은 `ToOne` 관계는 fetch join을 써도 데이터 row 수가 거의 늘어나지 않는다.

  예를 들어:

    ```
    MemberFood ->Food
    ```

  `MemberFood` 하나는 `Food` 하나만 가진다.

    ```
    MemberFood 1개 조회
    → Food 1개 붙음
    → 결과 row도 1개
    ```

  그래서 이런 건 안전하다.

    ```java
    select mf
    from MemberFood mf
    join fetch mf.food
    where mf.member in :members
    ```

  즉, `ToOne` 관계는 fetch join으로 N+1을 해결하기 좋다.
    
  ---

  ## 2. ToMany에는 fetch join을 조심해야 한다

  `OneToMany`, `ManyToMany` 같은 `ToMany` 관계는 fetch join을 하면 데이터가 뻥튀기될 수 있다.

  예를 들어 회원 1명이 음식 3개를 가진다고 하자.

    ```
    Member A
    - Food 1
    - Food 2
    - Food 3
    ```

  이걸 fetch join하면 SQL 결과는 이렇게 나온다.

    ```
    Member A + Food 1
    Member A + Food 2
    Member A + Food 3
    ```

  자바 객체로 보면 회원은 1명인데, DB row는 3줄이 된다.

  이게 데이터 뻥튀기다.
    
  ---

  ## 3. 데이터 뻥튀기란?

  원래 조회하고 싶은 루트 엔티티는 `Member` 1개인데, 연관된 컬렉션 개수만큼 SQL 결과 row가 늘어나는 현상이다.

    ```java
    select m
    from Member m
    join fetch m.memberFoodList
    ```

  결과가 이렇게 될 수 있다.

    ```
    member_id | member_name | food_id
    1         | Kim         | 10
    1         | Kim         | 11
    1         | Kim         | 12
    2         | Lee         | 13
    2         | Lee         | 14
    ```

  실제 회원은 2명이다.

  하지만 SQL row는 5개다.

    ```
    Kim → 음식 3개라서 3줄
    Lee → 음식 2개라서 2줄
    ```

  그래서 페이징이 깨질 수 있다.

    ```java
    PageRequest.of(0,2)
    ```

  이렇게 2명만 가져오고 싶었는데, SQL row 기준으로 2줄을 자르면

    ```
    Kim + Food 10
    Kim + Food 11
    ```

  이렇게 Kim만 나오고 Lee는 안 나올 수 있다.

  그래서 Hibernate는 컬렉션 fetch join + pagination을 위험하게 본다.
    
  ---

  ## 4. 그래서 ToMany는 batch size를 많이 쓴다

  `Member -> MemberFood`는 `OneToMany`다.

  이 경우에는 보통 Member를 먼저 페이징 조회한다.

    ```java
    List<Member>members = memberRepository.findAllBy(pageable);
    ```

  그 다음 컬렉션을 사용할 때 batch fetching으로 한 번에 가져온다.

    ```
    Member 1의 memberFoodList 조회
    Member 2의 memberFoodList 조회
    Member 3의 memberFoodList 조회
    ```

  원래는 이렇게 N번 나갈 수 있는 쿼리를

    ```java
    select *
    from member_food
    where member_id in (?, ?, ?)
    ```

  이런 식으로 묶어서 조회한다.

  즉,

    ```
    ToMany fetch join X
    ToMany batch size O
    ```

  가 보통 안전한 전략이다.
    
  ---

  ## 5. ToOne에는 batch size를 안 써도 되는가?

  정확히 말하면 **안 써도 된다기보다는 fetch join이 더 자주 쓰인다**가 맞다.

  `ToOne`도 지연 로딩이면 N+1이 생길 수 있다.

    ```java
    memberFood.getFood().getName()
    ```

  이때 `Food`를 하나씩 조회하면 N+1이 발생한다.

  해결 방법은 둘 다 가능하다.

    ```
    1. fetch join 사용
    2. batch size 사용
    ```

  하지만 `ToOne`은 fetch join을 해도 데이터 뻥튀기가 없으므로 보통 fetch join을 많이 쓴다.

  정리하면:

    ```
    ToOne = fetch join 권장
    ToMany = batch size 권장
    ```
    
  ---

  ## 6. fetch join은 모든 컬럼을 가져온다

  fetch join은 엔티티를 조회하는 방식이다.

    ```java
    select m
    from Member m
    join fetch m.memberFoodList
    ```

  이렇게 조회하면 `Member` 엔티티와 `MemberFood` 엔티티를 영속성 컨텍스트에 올려야 한다.

  그래서 연관 엔티티의 필요한 컬럼만 가져오는 것이 아니라, 엔티티를 구성하는 컬럼을 가져온다.

  즉, 이런 상황에서는 비효율적일 수 있다.

    ```
    Food의 name만 필요하다
    그런데 Food의 모든 컬럼을 조회한다
    ```

  이럴 때 DTO Projection을 사용한다.

    ```java
    select new com.example.MemberFoodDto(
        m.id,
        m.name,
        f.id,
        f.name
    )
    from MemberFood mf
    join mf.member m
    join mf.food f
    where m in :members
    ```

  DTO Projection은 필요한 컬럼만 직접 고르는 방식이다.
    
  ---

  ## 7. fetch join에서 조건절을 못 쓰는 건가?

  완전히 못 쓰는 건 아니다.

  일반적인 `where` 조건은 사용할 수 있다.

    ```java
    select mf
    from MemberFood mf
    join fetch mf.food f
    where mf.member in :members
    ```

  이건 가능하다.

  다만 조심해야 하는 건 **컬렉션 fetch join에 조건을 걸어서 일부만 가져오는 것**이다.

  예를 들어:

    ```java
    select m
    from Member m
    join fetch m.memberFoodList mf
    where mf.food.name = '치킨'
    ```

  이렇게 하면 `Member`의 `memberFoodList` 전체가 아니라 조건에 맞는 일부만 채워질 수 있다.

  그런데 JPA 입장에서는 이 컬렉션이 “로딩 완료된 컬렉션”처럼 보일 수 있어서 위험하다.

  그래서 컬렉션을 조건으로 필터링해서 필요한 데이터만 가져오고 싶다면 fetch join보다는 DTO Projection이 더 안전하다.
    
  ---

  ## 8. 코드 흐름

    ```java
    List<Member>members=memberRepository.findAllBy(pageable);
    ```

  먼저 `Member`만 페이징 조회한다.

    ```
    회원 2명만 정확히 조회
    ```

  그 다음:

    ```java
    List<MemberFood> memberFoods =
        memberFoodRepository.findAllByMemberInWithFood(members);
    ```

  `MemberFood`를 따로 조회하면서 `Food`는 fetch join한다.

    ```java
    select mf
    from MemberFood mf
    join fetch mf.food
    where mf.member in :members
    ```

  이 구조는 다음과 같다.

    ```
    Member -> MemberFood = ToMany라서 따로 조회
    MemberFood -> Food = ToOne이라서 fetch join
    ```

  그래서 페이징도 안전하고, Food N+1도 막을 수 있다.
    
  ---

  ## 9. 최종 정리

    ```
    N+1 해결 = 무조건 fetch join이 아니다
    ```

  관계에 따라 다르게 봐야 한다.

    ```
    ManyToOne, OneToOne
    = ToOne 관계
    = fetch join 사용해도 데이터 뻥튀기 적음
    = fetch join 권장
    ```

    ```
    OneToMany, ManyToMany
    = ToMany 관계
    = fetch join 시 row 증가
    = 페이징 깨질 수 있음
    = batch size 또는 별도 조회 권장
    ```

    ```
    DTO Projection
    = 엔티티 전체가 아니라 필요한 컬럼만 조회할 때 사용
    = 조회 전용 화면, API 응답 최적화에 적합
    ```

  한 줄로 정리하면:

  `ToOne은 fetch join으로 가져오고, ToMany는 페이징을 먼저 한 뒤 batch size나 별도 조회로 가져오며, 필요한 컬럼만 있으면 DTO Projection을 쓰는 것이 좋다.`


- Hibernate 2차 캐시

  ## 1. 1차 캐시

  Hibernate의 `1차 캐시`는 `Persistence Context` 안에 있다.

  즉, 하나의 트랜잭션 안에서만 동작한다.

    ```
    Membermember1=em.find(Member.class,1L);
    Membermember2=em.find(Member.class,1L);
    ```

  같은 트랜잭션 안에서 같은 엔티티를 두 번 조회하면, 두 번째는 DB에 가지 않고 1차 캐시에서 가져온다.
    
  ---

  ## 2. 2차 캐시

  `2차 캐시`는 여러 Session, 여러 트랜잭션이 함께 사용할 수 있는 캐시다.

    ```
    1차 캐시: Persistence Context 내부
    2차 캐시: SessionFactory 단위의 공용 캐시
    ```

  즉, 한 번 조회한 엔티티를 2차 캐시에 저장해두면, 다음 트랜잭션에서 같은 엔티티를 조회할 때 DB를 다시 조회하지 않고 캐시에서 가져올 수 있다.
    
  ---

  ## 3. 조회 흐름

  엔티티를 조회할 때 흐름은 보통 다음과 같다.

    ```
    1. 1차 캐시에서 찾는다
    2. 없으면 2차 캐시에서 찾는다
    3. 2차 캐시에도 없으면 DB에서 조회한다
    4. DB에서 가져온 결과를 1차 캐시와 2차 캐시에 저장한다
    ```

  예를 들면 다음과 같다.

    ```
    Membermember=em.find(Member.class,1L);
    ```

  처음 조회할 때는 DB에 간다.

    ```
    DB 조회 → 1차 캐시 저장 → 2차 캐시 저장
    ```

  다른 트랜잭션에서 다시 조회하면

    ```
    1차 캐시 없음 → 2차 캐시 있음 → DB 조회 생략
    ```

  이렇게 동작할 수 있다.
    
  ---

  ## 4. 2차 캐시를 쓰는 이유

  DB 조회 횟수를 줄이기 위해 사용한다. 특히 자주 조회되지만 잘 변경되지 않는 데이터에 적합하다.

  예를 들면 다음과 같다.

    ```
    카테고리
    지역 코드
    권한 정보
    설정값
    상품 기본 정보
    공지사항 상세
    ```

  이런 데이터는 매번 DB에서 조회할 필요가 없으므로 캐싱 효과가 크다.
    
  ---

  ## 5. 2차 캐시가 적합하지 않은 경우

  자주 변경되는 데이터에는 조심해야 한다.

  예를 들면 다음과 같다.

    ```
    재고
    잔액
    주문 상태
    실시간 좋아요 수
    조회수
    ```

  이런 데이터는 캐시에 오래 남아 있으면 실제 DB 값과 달라질 수 있다.

  즉, 2차 캐시는 **읽기 빈도가 높고 변경 빈도가 낮은 데이터**에 적합하다.
    
  ---

  ## 6. 1차 캐시와 2차 캐시 차이

  | 구분 | 1차 캐시 | 2차 캐시 |
      | --- | --- | --- |
  | 위치 | Persistence Context | SessionFactory |
  | 범위 | 하나의 트랜잭션 | 애플리케이션 전체 |
  | 기본 사용 여부 | 기본 사용 | 별도 설정 필요 |
  | 공유 여부 | 공유 안 됨 | 여러 Session이 공유 |
  | 목적 | 같은 트랜잭션 내 중복 조회 방지 | 트랜잭션 간 DB 조회 감소 |
    
  ---

  ## 7. 사용 방법

  2차 캐시는 기본적으로 꺼져 있다.

  사용하려면 설정과 캐시 제공자가 필요하다.

  대표적으로 Ehcache, Caffeine, Redis 같은 캐시 저장소를 사용할 수 있다.

  그리고 엔티티에 캐시 설정을 붙인다.

    ```java
    @Entity
    @Cacheable
    @org.hibernate.annotations.Cache(
        usage = CacheConcurrencyStrategy.READ_ONLY
    )
    public class Category {
    
        @Id
        private Long id;
    
        private String name;
    }
    ```

  이렇게 하면 `Category` 엔티티는 2차 캐시 대상이 될 수 있다.
    
  ---

  ## 8. CacheConcurrencyStrategy

  2차 캐시에서는 동시성 전략을 정해야 한다.

  ### READ_ONLY

  절대 수정되지 않는 데이터에 사용한다.

    ```
    @Cache(usage=CacheConcurrencyStrategy.READ_ONLY)
    ```

  예를 들어 코드성 데이터, 카테고리 등에 적합하다.
    
  ---

  ### NONSTRICT_READ_WRITE

  가끔 수정되지만, 약간의 데이터 불일치를 허용할 수 있을 때 사용한다.

    ```
    @Cache(usage=CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
    ```

  정확성이 아주 중요하지 않은 조회성 데이터에 사용할 수 있다.
    
  ---

  ### READ_WRITE

  수정도 일어나고, 캐시 일관성도 어느 정도 보장해야 할 때 사용한다.

    ```
    @Cache(usage=CacheConcurrencyStrategy.READ_WRITE)
    ```

  하지만 그만큼 비용이 더 든다.
    
  ---

  ### TRANSACTIONAL

  트랜잭션 기반 캐시 일관성이 필요한 경우 사용한다.

  일반적인 프로젝트에서는 자주 쓰지 않는다.
    
  ---

  ## 9. 주의할 점

  2차 캐시는 무조건 성능을 올려주는 기능이 아니다.

  캐시 저장, 조회, 무효화 비용이 있기 때문이다.

  또한 데이터가 자주 변경되면 캐시 일관성 관리가 어려워진다.

  그래서 아무 엔티티에나 적용하면 오히려 성능이 나빠질 수 있다.

- 부록2 정리

  ## QueryDSL이란?

  `QueryDSL`은 **자바 코드로 SQL 또는 JPQL 쿼리를 작성할 수 있게 해주는 기술**이다.

  기존 JPQL은 문자열로 쿼리를 작성한다.

    ```
    @Query("select m from Member m where m.name = :name")
    List<Member>findByName(Stringname);
    ```

  하지만 QueryDSL은 문자열이 아니라 코드로 작성한다.

    ```
    queryFactory
    .selectFrom(member)
    .where(member.name.eq(name))
    .fetch();
    ```

  즉, QueryDSL은 **문자열 쿼리의 단점을 줄이고, 타입 안전한 쿼리를 작성하기 위해 사용한다.**
    
  ---

  ## Fluent API란?

  `Fluent API`는 메서드를 이어 붙이면서 읽기 쉽게 코드를 작성하는 방식이다.

    ```
    queryFactory
    .selectFrom(member)
    .where(member.name.eq("kim"))
    .orderBy(member.createdAt.desc())
    .fetch();
    ```

  코드가 문장처럼 이어지기 때문에 쿼리 구조를 파악하기 쉽다.

    ```
    member에서 조회하고
    name이 kim인 데이터만 가져오고
    createdAt 기준 내림차순 정렬하고
    결과를 가져온다
    ```
    
  ---

  ## 단순 문자열 쿼리의 단점

  문자열 쿼리는 오타가 있어도 컴파일 시점에 잡기 어렵다.

    ```
    @Query("select m from Member m where m.namme = :name")
    ```

  여기서 `namme`는 오타다.

  하지만 문자열이기 때문에 컴파일 단계에서는 잘못된 필드명인지 알기 어렵다.

  실행 시점이 되어야 오류가 발생할 수 있다.
    
  ---

  ## Fluent API를 사용하는 장점

  ### 1. IDE 코드 자동 완성을 사용할 수 있다

  QueryDSL은 자바 코드로 쿼리를 작성하기 때문에 IDE 자동 완성을 사용할 수 있다.

    ```
    member.name
    member.email
    member.createdAt
    ```

  이런 식으로 엔티티 필드를 직접 참조할 수 있다.
    
  ---

  ### 2. 문법적으로 잘못된 쿼리를 줄일 수 있다

  문자열 쿼리는 오타나 잘못된 문법을 실행 전까지 모를 수 있다.

  하지만 QueryDSL은 코드 기반이기 때문에 잘못된 필드 접근이나 타입 오류를 컴파일 단계에서 어느 정도 잡을 수 있다.

    ```
    member.name.eq(10)
    ```

  `name`이 문자열인데 숫자와 비교하려고 하면 타입이 맞지 않아 오류가 날 수 있다.
    
  ---

  ### 3. 도메인 타입과 프로퍼티를 안전하게 참조할 수 있다

  QueryDSL은 `Q클래스`를 통해 엔티티의 필드를 타입 안전하게 참조한다.

    ```
    QMembermember=QMember.member;
    ```

    ```
    member.name.eq("kim")
    member.age.goe(20)
    ```

  이렇게 실제 도메인 필드를 기반으로 쿼리를 작성한다.
    
  ---

  ### 4. 리팩토링에 유리하다

  예를 들어 `Member` 엔티티의 필드명이 바뀌었다고 하자.

    ```
    privateStringname;
    ```

  이걸

    ```
    privateStringnickname;
    ```

  으로 변경하면 문자열 JPQL은 직접 찾아서 수정해야 한다.

    ```
    "where m.name = :name"
    ```

  하지만 QueryDSL은 코드 참조이기 때문에 IDE의 리팩토링 기능으로 함께 변경하거나, 잘못된 참조를 컴파일 오류로 확인할 수 있다.
    
  ---

  ## 동적 쿼리가 필요한 이유

  검색 화면에서는 사용자가 어떤 필터를 선택할지 모른다.

  예를 들어 음식점을 검색한다고 하면 조건이 매번 달라질 수 있다.

    ```
    지역만 선택
    지역 + 카테고리 선택
    지역 + 카테고리 + 가격대 선택
    지역 + 별점 선택
    아무 조건도 선택하지 않음
    ```

  정적 쿼리는 쿼리 구조가 고정되어 있다.

    ```
    @Query("""
    select r
    from Restaurant r
    where r.region = :region
    and r.category = :category
    and r.priceRange = :priceRange
    """)
    ```

  이 경우 사용자가 `region`만 선택했을 때, 나머지 조건을 어떻게 처리할지 복잡해진다.

  반면 동적 쿼리는 조건이 들어온 경우에만 where 절을 추가할 수 있다.

    ```
    region이 있으면 region 조건 추가
    category가 있으면 category 조건 추가
    priceRange가 있으면 priceRange 조건 추가
    ```

  그래서 검색 필터, 관리자 목록 조회, 정렬 조건이 많은 API에서는 동적 쿼리가 필요하다.
    
  ---

  ## QueryDSL 사용 흐름

  QueryDSL은 보통 다음 순서로 사용한다.

    ```
    1. Q클래스를 생성한다
    2. JPAQueryFactory로 쿼리를 작성한다
    3. BooleanBuilder 또는 BooleanExpression으로 조건을 조립한다
    4. fetch()로 결과를 조회한다
    ```
    
  ---

  ## 1. Q클래스란?

  `Q클래스`는 QueryDSL이 엔티티를 기반으로 자동 생성하는 클래스다.

  예를 들어 이런 엔티티가 있으면

    ```
    @Entity
    publicclassMember {
    
        @Id
    privateLongid;
    
    privateStringname;
    
    privateIntegerage;
    }
    ```

  QueryDSL은 대략 이런 Q클래스를 만든다.

    ```
    QMembermember=QMember.member;
    ```

  그리고 이 Q클래스를 통해 필드에 접근한다.

    ```
    member.id
    member.name
    member.age
    ```

  즉, Q클래스는 **QueryDSL에서 엔티티 필드를 코드로 안전하게 사용하기 위한 클래스**다.
    
  ---

  ## 2. 코드로 쿼리문 작성

  QueryDSL에서는 SQL이나 JPQL을 문자열로 직접 쓰지 않고 메서드 체인으로 작성한다.

    ```
    List<Member>result=queryFactory
    .selectFrom(member)
    .where(member.age.goe(20))
    .orderBy(member.name.asc())
    .fetch();
    ```

  의미는 다음과 같다.

    ```
    select*
    from member
    where age>=20
    orderby nameasc
    ```
    
  ---

  ## 3. BooleanBuilder로 Where문 작성

  `BooleanBuilder`는 조건을 하나씩 추가하면서 동적 where 절을 만들 때 사용한다.

  예를 들어 검색 조건 DTO가 있다고 하자.

    ```
    publicrecordMemberSearchCondition(
    Stringname,
    IntegerminAge,
    IntegermaxAge
    ) {
    }
    ```

  조건이 들어온 경우에만 where 절에 추가할 수 있다.

    ```
    publicList<Member>search(MemberSearchConditioncondition) {
    BooleanBuilderbuilder=newBooleanBuilder();
    
    if (condition.name()!=null) {
    builder.and(member.name.contains(condition.name()));
        }
    
    if (condition.minAge()!=null) {
    builder.and(member.age.goe(condition.minAge()));
        }
    
    if (condition.maxAge()!=null) {
    builder.and(member.age.loe(condition.maxAge()));
        }
    
    returnqueryFactory
    .selectFrom(member)
    .where(builder)
    .fetch();
    }
    ```

  예를 들어 `name = "kim"`만 들어오면

    ```
    where namelike'%kim%'
    ```

  `minAge = 20`, `maxAge = 30`도 들어오면

    ```
    where namelike'%kim%'
    and age>=20
    and age<=30
    ```

  이렇게 조건이 동적으로 붙는다.
    
  ---

  ## BooleanBuilder 흐름

    ```
    처음에는 빈 조건이다
    name이 있으면 name 조건을 추가한다
    minAge가 있으면 최소 나이 조건을 추가한다
    maxAge가 있으면 최대 나이 조건을 추가한다
    최종 builder를 where에 넣는다
    ```

  즉, 사용자가 선택한 필터만 쿼리에 반영할 수 있다.
    
  ---

  ## 예시: 음식 검색

    ```
    publicList<Food>searchFood(FoodSearchConditioncondition) {
    BooleanBuilderbuilder=newBooleanBuilder();
    
    if (condition.category()!=null) {
    builder.and(food.category.eq(condition.category()));
        }
    
    if (condition.keyword()!=null) {
    builder.and(food.name.contains(condition.keyword()));
        }
    
    if (condition.minPrice()!=null) {
    builder.and(food.price.goe(condition.minPrice()));
        }
    
    if (condition.maxPrice()!=null) {
    builder.and(food.price.loe(condition.maxPrice()));
        }
    
    returnqueryFactory
    .selectFrom(food)
    .where(builder)
    .orderBy(food.createdAt.desc())
    .fetch();
    }
    ```

  이렇게 하면 검색 조건이 몇 개 들어오든 유연하게 처리할 수 있다.
    
  ---

  ## 정리

  QueryDSL은 **문자열이 아니라 자바 코드로 쿼리를 작성하는 방식**이다.

  Fluent API를 사용하기 때문에 다음 장점이 있다.

    ```
    IDE 자동 완성을 사용할 수 있다
    잘못된 필드명이나 타입 오류를 컴파일 단계에서 잡기 쉽다
    도메인 타입과 프로퍼티를 안전하게 참조할 수 있다
    리팩토링에 유리하다
    동적 쿼리를 작성하기 좋다
    ```

  그리고 사용 흐름은 다음과 같다.

    ```
    Q클래스 생성
    JPAQueryFactory로 쿼리 작성
    BooleanBuilder로 조건 조립
    fetch로 결과 조회
    ```

- 배치 사이즈

  ## 배치 사이즈 (Batch Size)

  `배치 사이즈(batch size)`는 연관 엔티티를 조회할 때 여러 개를 한 번에 묶어서 조회하는 기능이다.

  주로 `N+1 문제`를 줄이기 위해 사용한다.

  예를 들어 회원 여러 명을 조회했다고 하자.

    ```
    List<Member>members=memberRepository.findAll();
    ```

  그리고 각 회원의 주문 목록을 조회하면

    ```
    members.forEach(member -> {
    member.getOrders().size();
    });
    ```

  기본적으로는 이렇게 된다.

    ```
    회원 조회 1번
    회원1 orders 조회
    회원2 orders 조회
    회원3 orders 조회
    ...
    ```

  즉:

    ```
    1 + N 문제 발생
    ```
    
  ---

  ### batch size 적용

    ```
    @OneToMany(mappedBy="member")
    @BatchSize(size=100)
    privateList<Order>orders;
    ```

  또는 글로벌 설정:

    ```
    spring:
      jpa:
        properties:
          hibernate.default_batch_fetch_size: 100
    ```

  이렇게 하면 Hibernate가 여러 member_id를 모아서 한 번에 조회한다.

    ```
    select*
    from orders
    where member_idin (?, ?, ?, ...)
    ```

  즉:

    ```
    회원 조회 1번
    orders IN 조회 1번
    ```

  으로 줄어든다.
    
  ---

  ### 언제 많이 사용하는가

  보통 `ToMany 관계`에서 많이 사용한다.

    ```
    OneToMany
    ManyToMany
    ```

  왜냐하면 ToMany fetch join은 데이터 뻥튀기와 페이징 문제가 생길 수 있기 때문이다.

  그래서:

    ```
    ToOne → fetch join
    ToMany → batch size
    ```

  전략을 자주 사용한다.

- transform - groupBy

  # transform - groupBy

  QueryDSL에서 DTO를 계층 구조처럼 묶을 때 사용하는 기능이다.

  예를 들어 DB 결과는 flat한 row 형태다.

    ```
    member_id | member_name | food_name
    1         | Kim         | Pizza
    1         | Kim         | Chicken
    2         | Lee         | Burger
    ```

  하지만 실제로 원하는 구조는 이런 형태일 수 있다.

    ```
    Kim
     - Pizza
     - Chicken
    
    Lee
     - Burger
    ```
    
  ---

  ## QueryDSL transform + groupBy

    ```
    Map<Long,List<String>>result=
    queryFactory
    .from(memberFood)
    .join(memberFood.member,member)
    .join(memberFood.food,food)
    .transform(
    GroupBy.groupBy(member.id)
    .as(GroupBy.list(food.name))
            );
    ```
    
  ---

  ## 역할

    ```
    DB row 결과를
    자바 컬렉션 구조로 그룹핑하는 기능
    ```

  즉 SQL의 `GROUP BY`와는 조금 다르다.

  SQL group by는 집계용이다.

    ```
    groupby member_id
    count(*)
    ```

  하지만 QueryDSL의 transform-groupBy는:

    ```
    조회 결과를 자바 Map/List 구조로 묶는 기능
    ```

  이다.
    
  ---

  ## 자주 쓰는 이유

  DTO 계층 구조 만들 때 편하다.

  예:

    ```
    [
      {
        "memberId":1,
        "foods": ["Pizza","Chicken"]
      }
    ]
    ```

  이런 응답 만들 때 사용한다.

- order by null

  # order by null

  `order by null`은 MySQL 등에서 불필요한 정렬을 제거하기 위해 사용하는 문법이다.
    
  ---

  ## 왜 사용하는가

  예를 들어 `group by`를 사용하면 MySQL은 내부적으로 정렬을 수행할 수 있다.

    ```
    select member_id,count(*)
    from orders
    groupby member_id
    ```

  이때 정렬 비용이 발생할 수 있다.
    
  ---

  ## order by null 추가

    ```
    select member_id,count(*)
    from orders
    groupby member_id
    orderbynull
    ```

  이렇게 하면:

    ```
    정렬하지 마라
    ```

  라는 의미가 된다.
    
  ---

  ## 효과

    ```
    불필요한 sort 제거
    filesort 제거 가능
    group by 성능 개선 가능
    ```

  특히 MySQL에서 예전 버전 최적화에 자주 등장했다.

- 카테시안 곱

  # 카테시안 곱 (Cartesian Product)

  카테시안 곱은 조인 조건 없이 모든 경우의 수가 곱해지는 현상이다.

  예를 들어:

    ```
    회원 3명
    음식 4개
    ```

  조인 조건 없이 합치면:

    ```
    3 × 4 = 12 row
    ```

  가 생성된다.
    
  ---

  ## 예시

    ```
    select*
    from member, food
    ```

  또는 잘못된 join:

    ```
    select*
    from member m
    join food f
    ```

  조건이 없으면 모든 row 조합이 만들어진다.
    
  ---

  ## 문제점

    ```
    row 수 폭발
    메모리 사용 증가
    성능 급격히 저하
    ```

  특히 fetch join 여러 개를 잘못 사용하면 카테시안 곱이 발생할 수 있다.
    
  ---

  ## fetch join에서의 예

    ```
    selectm
    fromMemberm
    joinfetchm.orders
    joinfetchm.reviews
    ```

  만약:

    ```
    orders 3개
    reviews 4개
    ```

  면:

    ```
    3 × 4 = 12 row
    ```

  가 된다.

  즉 컬렉션끼리 서로 곱해진다.

- MultipleBagFetchException

  # MultipleBagFetchException

  Hibernate에서 발생하는 대표적인 fetch join 예외다.
    
  ---

  ## Bag이란?

  Hibernate에서 `List` 컬렉션을 Bag으로 본다.

  예:

    ```
    @OneToMany
    privateList<Order>orders;
    
    @OneToMany
    privateList<Review>reviews;
    ```

  둘 다 Bag이다.
    
  ---

  ## 문제 상황

    ```
    selectm
    fromMemberm
    joinfetchm.orders
    joinfetchm.reviews
    ```

  즉:

    ```
    List 컬렉션 두 개를 동시에 fetch join
    ```

  하려고 하면 발생할 수 있다.
    
  ---

  ## 왜 발생하는가

  DB row는 flat하다.

  예:

    ```
    회원
     - 주문 3개
     - 리뷰 4개
    ```

  fetch join하면:

    ```
    3 × 4 = 12 row
    ```

  가 된다.

  Hibernate 입장에서는:

    ```
    어떤 row가 어떤 컬렉션 데이터인지
    정확히 복원하기 어려움
    ```

  문제가 생긴다.

  그래서 아예 예외를 발생시킨다.

    ```
    MultipleBagFetchException
    ```
    
  ---

  ## 해결 방법

  ### 1. 하나만 fetch join

    ```
    joinfetchm.orders
    ```

  나머지는 batch size 사용.
    
  ---

  ### 2. List 대신 Set 사용

    ```
    privateSet<Order>orders;
    ```

  Hibernate가 중복 제거를 조금 더 안전하게 처리할 수 있다.

  하지만 무조건 해결책은 아니다.
    
  ---

  ### 3. batch fetching 사용

  실무에서 가장 많이 쓰는 방식이다.

    ```
    ToOne → fetch join
    ToMany → batch size
    ```