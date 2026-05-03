- 히카리CP 작동원리

  참고자료: https://justdo1tme.tistory.com/entry/Spring-Boot-DB-Connection-Pool%EA%B3%BC-Hikari-CP-%EC%84%A4%EC%A0%95

    - 개념

      JDBC는 Java Database Connectivity의 약자로 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다.
      히카리CP(HikariCP)는 자바에서 사용하는 JDBC 커넥션 풀(Connection Pool) 라이브러리다.

    - 작동 원리

      Thread가 커넥션을 요청하면 유휴 커넥션을 찾아 반환한다.
      여기서 이전에 사용했던 커넥션이 존재하는 경우, 이를 우선적으로 반환한다.
      가능한 커넥션이 존재하지 않으면, HandOffQueue를 Polling하면서 다른 Thread가 커넥션을 반납하기를 기다린다.
      다른 Thread가 커넥션을 반납하면, 커넥션 풀이 커넥션 사용 내역을 기록하고 HandOffQueue에 반납된 커넥션을 삽입한다.
      이를 통해 HandOffQueue를 Polling하던 Thread는 커넥션을 획득한다.

- SQM과 PartTree
    - 개념

      PartTree는 Spring Data JPA에서 메서드 이름 기반 쿼리를 분석하기 위해 사용하는 구조다.

      예를 들어, `List<Member> findByNameAndAgeGreaterThan(String name, int age);` 와 같은 메서드가 있을 때 PartTree는 find, by, name, and, age, greaterthan으로 쪼갠 후 분석하여 조건, 정렬, 제한 개수 등 쿼리 조건으로 해석한다.

      SQM(Semantic Query Model)은 Hibernate가 JPQL을 해석해서 만드는 중간 쿼리 모델이다.

    - 전체 흐름

      Repository 코드:

        ```java
        public interface MemberRepository extends JpaRepository<Member, Long> {
        
            List<Member> findByNameAndAgeGreaterThan(String name, int age);
        }
        ```

      Spring Data JPA PartTree 해석 결과:

        ```
        name = ?
        AND
        age > ?
        ```

      생성되는 JPQL:

        ```java
        select m
        from Member m
        where m.name = :name
        and m.age > :age
        ```

      Hibernate:

        ```
        JPQL 분석 -> SQM 생성 -> SQL 생성
        ```

      최종 SQL:

        ```sql
        select *
        from member
        where name = ?
        and age > ?
        ```

- EntityHolder

  `EntityHolder`는 Hibernate가 **엔티티 인스턴스와 그 엔티티에 대한 관리 정보를 함께 들고 있는 구조이다.** 영속성 컨텍스트 내부에서 특정 `EntityKey`에 해당하는 엔티티 관련 정보를 묶어서 보관하는 역할을 한다.
  영속성 컨텍스트에서 엔티티를 구분할 때 사용하는 EntityKey는 엔티티 타입 + id을 저장한다.

    ```
    Persistence Context
        ↓
    EntityKey(Member, 1)
        ↓
    EntityHolder
        ├─ Member 실제 객체
        └─ EntityEntry
              ├─ status
              ├─ loadedState
              ├─ id
              ├─ version
              └─ lockMode
    ```

- EntityEntry
    - EntityEntry는 Hibernate가 **영속 상태의 엔티티 하나에 대해 관리하는 메타데이터**다.
    - EntityEntry가 관리하는 것
        - 엔티티 상태: 엔티티가 현재 어떤 상태인지 관리한다.

      예:

        ```
        MANAGED
        READ_ONLY
        DELETED
        GONE
        LOADING
        ```

      예를 들어:

        ```java
        Member member = entityManager.find(Member.class, 1L);
        ```

      이 경우 `member`는 영속 상태이므로 Hibernate 내부에서는 대략 `MANAGED` 상태로 관리된다.

        - 식별자 id

        ```
        member.getId();// 1L
        ```

      Hibernate는 이 엔티티가 어떤 DB row와 매핑되는지 알아야 한다. 그래서 `EntityEntry`는 엔티티의 id 정보를 가진다.

        - loadedState

      `loadedState`는 엔티티가 처음 조회되었을 때의 값이다.

      예를 들어 DB에 이런 데이터가 있었다고 하자.

        ```
        id = 1
        name = "kim"
        age = 20
        ```

      Hibernate는 최초 상태를 저장한다.

        ```
        loadedState = ["kim", 20]
        ```

      그 후 값을 바꾸면 현재 객체 상태:

        ```
        currentState = ["kim", 21]
        ```

      Hibernate는 flush 시점에 loadedState와 currentState를 비교한다.
      다르면 update SQL을 만든다.

        ```
        update member
        set age=21
        where id=1;
        ```

      즉, `EntityEntry`는 dirty checking에 중요한 역할을 한다.
  
      | 구분 | EntityHolder | EntityEntry |
      | --- | --- | --- |
      | 역할 | 엔티티 객체와 관련 정보를 담는 보관 구조 | 엔티티 상태를 기록하는 메타데이터 |
      | 중심 | “어떤 엔티티 객체를 들고 있는가” | “그 엔티티가 어떤 상태인가” |
      | 관련 정보 | entity instance, proxy, entry 등 | status, loadedState, id, version, lockMode 등 |
      | 사용 위치 | Persistence Context 내부 엔티티 저장/조회 | dirty checking, flush, 상태 관리 |